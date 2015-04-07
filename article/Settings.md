
Android 4.4以下：

1、设置->安全->屏幕锁定->PIN或密码

2、设置->安全->加密手机，此时界面上有如下提示：

<img src="/image/encryp_pre.png" />

3、加密手机后，再一次进入：设置->安全->屏幕锁定，此时可以看到除了PIN和密码选项之外，其他选项都是置灰的，如下：

<img src="/image/encryp_after.png" />


但是在4.4以上的版本中，发现手机加密之后，所有屏幕锁定选项都是可以选择的。

首先，研究下Settings中，将锁屏模式置灰的逻辑，在ChooseLockGeneric.java中，代码如下：

```
        /***
         * Disables preferences that are less secure than required quality.
         *
         * @param quality the requested quality.
         */
        private void disableUnusablePreferences(final int quality, MutableBoolean allowBiometric) {
           //省略部分代码
            for (int i = entries.getPreferenceCount() - 1; i >= 0; --i) {
                Preference pref = entries.getPreference(i);
                if (pref instanceof PreferenceScreen) {
                    final String key = ((PreferenceScreen) pref).getKey();
                    boolean enabled = true;
                    boolean visible = true;
                    //以下if-else分别对应6种锁屏模式，enabled标识该模式是否可用
                    if (KEY_UNLOCK_SET_OFF.equals(key)) {//无
                        enabled = quality <= DevicePolicyManager.PASSWORD_QUALITY_UNSPECIFIED;
                    } else if (KEY_UNLOCK_SET_NONE.equals(key)) {//滑动
                        enabled = quality <= DevicePolicyManager.PASSWORD_QUALITY_UNSPECIFIED;
                    } else if (KEY_UNLOCK_SET_BIOMETRIC_WEAK.equals(key)) {//该锁屏模式无视
                        enabled = quality <= DevicePolicyManager.PASSWORD_QUALITY_BIOMETRIC_WEAK ||
                                allowBiometric.value;
                        visible = weakBiometricAvailable; // If not available, then don't show it.
                    } else if (KEY_UNLOCK_SET_PATTERN.equals(key)) {//图片
                        enabled = quality <= DevicePolicyManager.PASSWORD_QUALITY_SOMETHING;
                    } else if (KEY_UNLOCK_SET_PIN.equals(key)) {//PIN
                        enabled = quality <= DevicePolicyManager.PASSWORD_QUALITY_NUMERIC;
                    } else if (KEY_UNLOCK_SET_PASSWORD.equals(key)) {//密码
                        enabled = quality <= DevicePolicyManager.PASSWORD_QUALITY_COMPLEX;
                    }
                    if (!visible || (onlyShowFallback && !allowedForFallback(key))) {
                        entries.removePreference(pref);
                    } else if (!enabled) {
                        pref.setSummary(R.string.unlock_set_unlock_disabled_summary);
                        pref.setEnabled(false);
                    }
                }
            }
        }
```
以上代码的if-else逻辑中，可以看到上图五种锁屏模式，**enabled标识该模式是否可用,quality与DevicePolicyManager的锁屏级别大小决定着enabled的值为true或者false**，于是追溯quality的设值，该类中最终决定quality值的代码如下：
```
        private int upgradeQualityForEncryption(int quality) {
            int encryptionStatus = mDPM.getStorageEncryptionStatus();
            boolean encrypted = (encryptionStatus == DevicePolicyManager.ENCRYPTION_STATUS_ACTIVE)
                    || (encryptionStatus == DevicePolicyManager.ENCRYPTION_STATUS_ACTIVATING);
            if (encrypted) {
                if (quality < CryptKeeperSettings.MIN_PASSWORD_QUALITY) {
                    quality = CryptKeeperSettings.MIN_PASSWORD_QUALITY;
                }
            }
            return quality;
        }
```
重要的逻辑在于此：
```
 encryptionStatus = mDPM.getStorageEncryptionStatus();
```
大概原理是：通过DevicePolicyManager获取当前的加密策略，如果取值为ENCRYPTION_STATUS_ACTIVE或ENCRYPTION_STATUS_ACTIVATING，则将quality赋值为CryptKeeperSettings.MIN_PASSWORD_QUALITY，查看其值，发现：
```
static final int MIN_PASSWORD_QUALITY = DevicePolicyManager.PASSWORD_QUALITY_NUMERIC;
```
**也就是符合条件的情况下，quality的值最终为PASSWORD_QUALITY_NUMERIC，即PIN加密的级别，再对应disableUnusablePreferences()的逻辑，此时只有PIN和密码的锁屏模式可用。**

于是脱离这个繁杂aosp环境，直接写个简单的demo测试一下，代码如下：
```
DevicePolicyManager mDPM = (DevicePolicyManager) getSystemService(Context.DEVICE_POLICY_SERVICE);
int encryptionStatus = mDPM.getStorageEncryptionStatus();
Log.e("Genius", "checkSecuritySettingsSufficient: encryptionStatus=" + encryptionStatus);
```

4.4以下的情况：
手机加密前,encryptionStatus=1
手机加密后,encryptionStatus=3

**4.4和4.4以上的情况：
无论手机加密前或是加密后，encryptionStatus恒等于1**

于是追查getStorageEncryptionStatus的源码：
在这个位置下：
```
 /frameworks/base/services/java/com/android/server/DevicePolicyManagerService.java

 private int getEncryptionStatus() {
        String status = SystemProperties.get("ro.crypto.state", "unsupported");
        if ("encrypted".equalsIgnoreCase(status)) {
            return DevicePolicyManager.ENCRYPTION_STATUS_ACTIVE;//该值为3
        } else if ("unencrypted".equalsIgnoreCase(status)) {
            return DevicePolicyManager.ENCRYPTION_STATUS_INACTIVE;//该值为1
        } else {
            return DevicePolicyManager.ENCRYPTION_STATUS_UNSUPPORTED;
        }
    }
```
等于1时，indicating that encryption is supported, but is not currently active.

等于3时，indicating that encryption is active.

因此需要再检测一下手机加密前后，框架层中，SystemProperties.get("ro.crypto.state", "unsupported")的取值。
