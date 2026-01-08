---
title: "Meross IoT Part II: Mobile App"
author: me
date: 2025-12-29 01:00:00
categories: [IoT]
tags: [":mob", ":re", iot, mobile, reverse-engineering]
render_with_liquid: false
description: In this part I will quickly go through the reverse engineering process of the mobile app provided by Meross.
image:
    path: /assets/img/posts/iot/meross-logo-p2.png
---

The initial point for this investigation was to quickly examine the mobile app (without running it) to identify potential challenges when inspecting the protocol designed by Meross.

There are several ways on how to start investigating a mobile application: for instance, static analysis can be done with a tool such as MobSF. However, in this case I was interested in the cloud protocol and not the security of the application. Therefore, the source code is everything we need.

## Android Application OSINT

The Android application served as the primary target due to the possibility to recover the Java source code representation. A tool like jadx can be used to calculate the representation of the source code based on the compiled smali instructions inside DEX files within the APK file.

> In some cases, *jadx* fails to recover the Java code and skips this *bad code*.
> However, most of the times you will want to see that extra code, because it
> commonly affects the juicy functions.
>
> ```console
> $ jadx -d ./out --show-bad-code /path/to/app.apk
> ```
{: .prompt-tip }

Luckily, the developers of the app didn't apply heavy obfuscation techniques to the
`meross` package, which made the whole process of reversing a lot easier.

```console
$ find ./sources/com/meross -type d
sources/com/meross
sources/com/meross/localEncrypt
sources/com/meross/model
sources/com/meross/model/widget
sources/com/meross/model/scene
sources/com/meross/model/config
sources/com/meross/model/marketing
sources/com/meross/model/weather
sources/com/meross/model/protocol
sources/com/meross/model/protocol/inputOutput
sources/com/meross/model/protocol/control
--- snip ---
```

Before we can start diving into the API more, we need an account. So let's examine the signup logic first.

---

### Account registration

Creating new accounts within the mobile app is done over the UI. Back in the beginning of 2025, the attributes were slightly different, but the process remains the same.

```java
public Observable signUp(String email, String password, String accountCountryCode, String vendor, String serviceTicket) {
    ArrayMap arrayMap = new ArrayMap();
    arrayMap.put("email", email); // previously named "user"
    arrayMap.put("password", SafeUtils.md5(password));
    arrayMap.put("encryption", 1);
    if (serviceTicket != null) {
        arrayMap.put("serviceTicket", serviceTicket);
    }
    arrayMap.put("vendor", vendor);
    arrayMap.put("accountCountryCode", accountCountryCode.toUpperCase());
    arrayMap.put("mobileInfo", HttpUtils.getMobileInfo());
    return ((RemoteApi) RemoteAgent.getInstance().getRemoteApi())
            .signUp(HttpUtils.defaultHeaders(), HttpUtils.toJson(arrayMap))
            .j(new ResultFunc1());
}
```
{: file="com/meross/http/Remote.java (end 2025)" }

Here we can already derive some important facts about the registration process:

1. Registration uses the *MD5 hash* of the password, not the cleartext string. Because MD5 should not be used for security purposes to verify cryptographic identity due to known collisions, it is inherently insecure to use MD5 for sign-up purposes.
2. The `serviceTicket` parameter (an actual verification code reveiced via email or SMS) is not required to signup.
3. The following data is collected about a user when registering using the mobile app:
   ```python
    class SignUpData(BaseModel):
        email: str
        password: str
        encryption: int
        vendor: str = "meross"
        accountCountryCode: str
        mobileInfo: MobileInfo

    class MobileInfo(BaseModel):
        resolution: str = "<width>*<height>"
        carrier: str
        deviceModel: str
        mobileOs: str = "Android"
        mobileOsVersion: str
        uuid: str = "<android_id><random_uuid>"
   ```
   {: file="Collected data displayed as API models" }

To sign up without the need of the mobile app, you can use one of my scripts in the
[meross-scripts](https://github.com/MatrixEditor/meross-scripts) repository.

```bash
mrs cloud auth signup --username '<email>' --password '<password>'
```

---

### Account login

The next step after registering a new account is obviously to login, In this case,
the api is as simple as sending the previously created authentication data to the
cloud.

```java
public Observable signIn(String email, String password, String mfaCode) {
    HashMap map = new HashMap();
    map.put("email", email);
    map.put("password", SafeUtils.md5(password));
    map.put("encryption", 1);
    map.put("accountCountryCode", LocaleManager.getInstance().getLocale().toUpperCase());
    map.put("mobileInfo", HttpUtils.getMobileInfo());
    map.put("agree", 1);
    if (!TextUtils.isEmpty(mfaCode)) {
        map.put("mfaCode", mfaCode);
    }
    Map mapF = HttpUtils.defaultHeaders();
    return doPost(r("/v1/Auth/signIn", mapF, HttpUtils.toJson(map), new TypeReference<ResultLogin>() { // from class: com.meross.http.MS2FARemote.2
    }), "/v1/Auth/signIn", 1, mapF, map);
}
```
{: file="com/meross/http/Remote.java (end 2025)" }

Even though the `mobileInfo` and `accountCountryCode` are given here, they are optional during the login. Using an MD5 hash to login implies that pass-the-hash
attacks are possible by design.

To login using my custom meross-scripts, just use the same username and password again.

```bash
# login with username/password
mrs cloud auth login -U '<email>' --password '<password>' --use-encryption

# login via pass-the-hash
mrs cloud auth login -U '<email>' --hash '<md5_hash>'
```

---

### Other interesting files

The process of reversing the rest of the Cloud API (and local) remains the same within the Android source code. There are some other details which are somewhat interesting at first glance.

* There seems to be a discovery protocol that can be used to identify devices within the local network. More details on this protocol, internally called "MRS HI", will be given in the firmware analysis part. (`UDPBaseDiscover` in `com.meross.meross.utils`)
  You can use the `"hi"` command from the meross-scripts repository to discover all
  devices:
  ```console
  $ mrs hi
  I : (mss210-us) Device 'mss210'
  I :  Software: 6.1.8
  I :  Hardware: 6.0.0
  I :  MAC     : aa:bb:cc:dd:ee:ff
  I :  Location: 10.13.37.2:80
  I :  UUID    : XXXX
  ```

* There are some "beta" firmware binaries located in the *assets* directory among
  other interesting files:
  ```console
  $  ls -la ./resources/com.meross.meross.apk/assets
total 1436
-rw-rw-r--  1   167 beta_firmware_update_config.json
-rw-rw-r--  1 30469 countries.json
drwxrwxr-x  2  4096 dexopt
drwxrwxr-x  4  4096 files
drwxrwxr-x  2  4096 gif
-rw-rw-r--  1 39212 kookong_all_info.json
-rw-rw-r--  1  3932 kookong_order_info.json
-rw-rw-r--  1  1830 msl120.json
-rw-rw-r--  1 52029 msl210-jp-v2-rc0119.bin
-rw-rw-r--  1  1636 msl210.json
-rw-rw-r--  1 40191 mss420f-us-v2-rc0105.bin
-rw-rw-r--  1 66103 mss425-us-v1-rc0110.bin
-rw-rw-r--  1 30034 mss426s_uuids.json
drwxrwxr-x  3  4096 product
-rw-rw-r--  1 53605 RegionJsonData.dat
  ```
  A quick look on these firmware binaries reveals that they are most likely encrypted
  or encoded with a custom format (*Blog post will be published on that*).
  Interestingly enough, the `mss426s_uuids.json`
  contains a list of seemingly valid device UUIDs **and** MAC addresses.
  ```json
  {
    "mss426s_uuids": [
        "1807114624231901234534298f1f0e8e",
        "1808172927483429083434298f16289d",
        "1808174524147329083434298f16289e",
        "...",
    ],
    "mss426f_macs": [
        "34298F1626FA",
        "34298F162421",
        "...",
    ]
  }
  ```
  {: file="resources/com.meross.meross.apk/assets/mss426s_uuids.json" }
* From a security perspective, allowing cleartext traffic (HTTP) in the
  `network_security_config.xml` does not follow best practices, but in this
  case does not affect calling the cloud API.


---

This time I've covered some source code analysis (OSINT) of the Android mobile application in order to unserstand the cloud API. We now know how to register
accounts without providing associated mobile device information. Next time I will
take a look on the device itself and an integration without the need of the cloud
backend of Meross.