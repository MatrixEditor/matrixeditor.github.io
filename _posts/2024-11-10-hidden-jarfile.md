---
layout: post
category: re
title: Google's hidden JAR in Android Applications
---

---
{: data-content=" summary "}

*Android applications that integrate the Google Play Services Ads library starting from version 10.0.0*
*include an encrypted JAR file containing an additional DEX file, which implements extra tracking.*
*Although this functionality was not triggered in any of the applications I examined, the presence*
*of this functionality raises some questions.*

<br>

While examining the decompiled Java code of a random Android application, I came across some
unusual encryption and decryption routines, along with a number of encrypted strings. Digging
deeper, I found a huge base64-encoded string:

```
LKUWgWkzisgs8+2GWY/EzJjqacF7VIkG9Ha+T8/c0Rv2WrdKu19V6/aqlLTu/u/jI7vON1i5NPPShQvA6QJ9WdT...
```

What caught my attention was that this encrypted string was embedded within a class from the
Google Play Services Ads library (which means you can see that code if you decompile any of the libraries from
[Maven - Play Services Ads](https://mvnrepository.com/artifact/com.google.android.gms/play-services-ads), starting
from version 10.0.0). After inspecting the encryption routine, I uncovered some interesting findings:

```
┌──────────────────────┐
│ encoded_data: string │
└──────────┬───────────┘
           │ base64.decode
           │
┌──────────▼───┬────────────────────────┐
│ iv: byte[16] │ encrypted_data: byte[] │
└──────────┬───┴────────────────┬───────┘
           │                    │
           ▼                    │
      AES/CBC/PKCS5 ◄───────────┘
           │
           │
┌──────────▼───┐
│ data: byte[] │
└──────────────┘
```

Upon decrypting the data, it became clear that it represented a JAR (Java Archive) file, which contained
a single file: *classes.dex*. Using the decryption method outlined earlier, I was able to extract the
following when attempting to decrypt an example string with Python:

```
['META-INF/', 'META-INF/MANIFEST.MF', 'classes.dex']
```

At this point, the intention behind this JAR file became clear. Some other interesting classes revealed
the workflow:

* The file is first decrypted using the method described above and temporarily saved to the */cache* or */dex* directory (private app files).
* Then, the only file inside, *classes.dex*, is extracted and loaded using an *InMemoryDexClassLoader*.
* Before resuming the DEX file gets deleted
* The classes contained within this DEX file are responsible for gathering additional tracking data—because, of course, that's what they do.

**Important:** Although the functionality to load extra classes is present, the Android application in
question did not actually load any additional DEX files. Why this functionality exists in the first place,
I have no idea.

---
{: data-content=" Additional Data Collection by Google "}

{% raw %}

| Name                              | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `MotionEvent`                     | Collects information about a given <code>MotionEvent</code>. By default, the following information is extracted: `elapsedTime`, `pressure`, `windowObscured`, `historicalPressure`, `x`, `y`, `touchMajor`, `historicalTouchMajor`, `toolType`, `deviceId` |
| `uptimeMillis`                    | Queries information about the uptime since last boot of the device.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `availableProcessors`             | The amount of available processors                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `date`                            | THe current date as a timestamp                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `screen:width/height/orientation` | The device's screen-width, height and orientation                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `battery:status/level`            | Information about the current battery status. The returned array contains information about the current battery status, charging level and temperature.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `model`                           | the constant according to the current model (either `NORMAL_DEVICE`, `ROOTED_DEVICE` or `EMULATOR_DEVICE`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `usbStatus`                       | Whether the device is connected via a USB-cable.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `adbEnabled`                      | Whether ADB is enabled                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `android-id`                      | Android-ID ([AAID](https://developer.android.com/training/articles/user-data-ids))                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `usesProxy`                       | Whether the running app is using a global HTTP-Proxy                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `processInfo`                     | Whether the app is running in foreground                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |


{% endraw %}

---
{: data-content=" Decryption keys? "}

I won't be that guy who tells you to search for the keys by yourself. Here's a complete list up
to version *21.4.X*:


| Version | JAR name | AES Key |
|---------|----------|---------|
| v10_0   | "1470286953684" | "t2PZ/+5cxs0OItzPKdPmkcFYX6IMfjTFHkZNA+hRNgo=" |
| v10_2   | "1478228129219" | "pWgSmgxREOizVsrpWzv2FevgkMRzEzPQ2R2fRa7gjO4=" |
| v11_0   | "1489418796403" | "QZZCt2ftWILMiOv/bx0NwH1VFPjOT+QCiqkEm96fZOY=" |
| v11_2   | "1493867303508" | "/gWMIQhNeeE0o9ImzFWAkrkA4LURH3SPZZB9Qi7zn08=" |
| v11_4   | "1496809943795" | "XDnJYBO2E96/jOXCxl3pm4VcW8g69dVlp14eaOLilDs=" |
| v11_6   | "1501670890290" | "FUnu12BLeA90PMRjjzllkVEPyqHD6uiYJ0wfE9HQOe8=" |
| v11_8   | "1505450608132" | "WKdn2zzE+pFOb2FrixdUDF+m9GVRaxGTq2U3/uOmGmE=" |
| v12_0   | "1510898742191" | "fmyYtvp6GdfV8aXECqySf5usPZLp4lFIXsdmCOa6f3I=" |
| v15_0   | "1521499837408" | "fFhi0cTZpyVQYwMwl7BCfa0fa6esmkRUaNr4ktfJYZ8=" |
| v16_0   | - | v15_0 |
| v17_0   | "1529567361524" | "qDz6YvDkhwdxUOtNXedEKNdh2XDWXqUECYckxUUtMRo=" |
| v17_1   | - | v17_0 |
| v17_2   | "1542658731108" | "gjATLq4PR4tBy0NKJBUs0hq7sitSgRlGcsdxPuImAoM=" |
| v18_0   | - | v17_2 |
| v18_1   | "1557357152169" | "L4Jx3nWfLteifbNgb1nlz1ddHDij7T6WG7cXq30PHqM=" |
| v18_2   | "1561154238473" | "uMy2E6ap9wVg3mVwKQNsrfJRJbtVQEp/VRd7Q09cmuU=" |
| v18_3   | "1570054248636" | "4V37Zv/fqUn78vx5Tt2zbOoOKYn7HiwHmwoLsVX89T8=" |
| v18_4   | - | v18_3 |
| v19_0   | "1571257279724" | "ZXQfmMb0WIh6YWGswslNgWXzCL3/RF6Ojd69jZM1GPs=" |
| v19_1   | "1582435991586" | "WxtxskzIWp7xb2ZhbqdUNS00sGJjYhs08Ug4usVoMAE=" |
| v19_2   | "1584479576572" | "IJNS0zozPMEhxshZHhAgFyrxN+YsYMK+YdGkDew63Ko=" |
| v19_3   | "1588462714860" | "9q95/1/ZSgXD7f6ulHIPUr8z7TrGmKA5+GWSXv/CYFA=" |
| v19_4   | - | v19_3 |
| v19_5   | "1596060835607" | "S0dK5C0YO8sTjhVyMGQOiXGsVVkG8T8dYSBak1Q84XU=" |
| v19_6   | "1598581401714" | "aYH2WgIueW3uUAtM9Jfb3Db35FHySfU4OZ5JZXgXCVg=" |
| v19_7   | "1608138930680" | "ll+nowuQKLxZSE4zpeTvUl3Gha6AS9UBIOMBB5g+5uQ=" |
| v19_8   | "1610724645094" | "pPUxBYyr76piI8i0eva67UkfRUCvzuFdlUmAk6Mi2Tw=" |
| v20_0   | - | v19_8 |
| v20_1   | "1613498354782" | "AQZlye0Qf6I1JwsO6u2s3ZPB9yudAuKGNAQ9qUeSY1g=" |
| v20_2   | "1616432909849" | "RV61Zx08QI+r0KCLhOeBrJPnsMi/yhd3p5E5I04HG2U=" |
| v20_3   | "1621276117097" | "O5+EI9qd857uJNhBPBY+hYh5U8lug4S2akyjrXXZBPw=" |
| v20_4   | "1624498498047" | "BcYDljg1B827gWFuo6QrhFDPNyXbfMHz+vF2qZ+sQXs=" |
| v20_5   | "1629828815138" | "NYpdto3gBV8HiZtFXi3NN2dSfPyfe2T+8tUnAUjRH8A=" |
| v20_6   | "1633031840514" | "BcJJ7m9GnDZ5QH3kvN4kRXKQduFKSe4hbLIA7qGtn8k=" |
| v21_0   | "1644353911296" | "RP5LQuE/2876zTvAb2rVm25QfjxwoRyidjQTLjf0RRc=" |
| v21_1   | "1651189566953" | "kMdUJlXzMwplT8jSHASgWSZqedBabCsM4bGGMxTrHLk=" |
| v21_2   | "1655145693758" | "3ti3ezo40ryM1w4gfDBvjiqMuHkyKLnrJqm+zFKeNdY=" |
| v21_3   | "1658186039475" | "HeBkX9XaSpC6sV82I6X2HUgm82vH8VhIWt26LGkrI3A=" |
| v21_4   | "1661804530683" | "Mr/If/sfOJZdBhXPwMTpWZgVNQcYf180jXHJjh6tWS8=" |
