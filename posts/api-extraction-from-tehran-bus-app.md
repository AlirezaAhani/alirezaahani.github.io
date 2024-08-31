---
title: API Extraction from Tehran Public Transport Application
layout: posts.liquid
is_draft: false
published_date: 2024-08-29 10:30:00 +0330
description: Extracting bus and subway ETA APIs from Tehran Public Transport
categories: ['Technology', 'Android']
---
### Introduction
A few years ago, while riding on a bus I saw an infographic about this application which could tell you the ETA for the buses in Tehran. Intrigued by this claim, I installed the app named [Tehran Public Transport](https://play.google.com/store/apps/details?id=com.neda.buseta) on my phone.  
  
The initial screen tells you to register using your phone number and a code sent to it using SMS. A very common pattern for applications in Iran. After that it took some time to download a database of all stations, and it finally opened.  
  
<figure>
  <img alt="Screenshot of application" src="/public/images/api-extraction-from-tehran-bus-app/Screenshot_Tehran_Public_Transport.png" style="max-height: 700px !important;"/>
  <figcaption>Showing میدان انقلاب (Code 4613) station  on routes پایانه معین, ETA for that station is 8 minutes</figcaption>
</figure>
  
The app has two different types of interface, but it is not relevant to this post.  
As far as I have tested and used this app, the ETA is pretty accurate, but it does perform very poor on some routes (Which the drivers could be to blame, but that's another discussion)  
  
### Motivation
Just curiosity! Also, the app claims to be able to do ETA for subway stations as well, but none of the subway stations are listed in the app. So I wanted to investigate that as well.  
I use the subway nearly every day twice, to get from home to other places, so the ETA could be useful.  
  
### Initial decompiling
For the first step, I decided to decompile the APK using [Jadx](https://github.com/skylot/jadx), to obtain the APK, I used [Evozi's APK downloader](https://apps.evozi.com/apk-downloader/).  
Here is the result of decompiling the APK:  
  
<figure>
  <img alt="Jadx decompile" src="/public/images/api-extraction-from-tehran-bus-app/Screenshot_jadx_gui.png"/>
  <figcaption>Jadx decompile result</figcaption>
</figure>
  
The tracking and ETA seems to be coming from [IranTracking](https://irantracking.com/) a company for GPS tracking of trucks and buses. Maybe they had a contract with the city?  
The code seems pretty clean, surprisingly unlike most apps I have decompiled, the code is not very obfuscated and easy to read.  
After some investigating, I found `com.irantracking.tehranbus.common.api` which could be interesting for our use.  
```java
package com.irantracking.tehranbus.common.api;
 
import com.irantracking.tehranbus.common.data.network.request.BusEtaRequest;
import com.irantracking.tehranbus.common.data.network.request.SubwayEtaRequest;
import com.irantracking.tehranbus.common.data.network.response.RouteStationData;
import java.util.List;
import kotlin.Metadata;
import org.jetbrains.annotations.NotNull;
import retrofit2.Call;
import retrofit2.http.Body;
import retrofit2.http.POST;
 
public interface EtaApi {
  @POST("BusStopETA")
  @NotNull
  Call<List<RouteStationData.BusRouteStationData>> getETA(@Body     @NotNull BusEtaRequest body);
  
  @POST("SubwayStationETA")
  @NotNull
  Call<List<RouteStationData.SubwayRouteStationData>> getSubwayETA(@Body @NotNull SubwayEtaRequest body);
}
```
Looks like they are using `retrofit2` for their API calls and transforming them into objects. Interestingly there is also a `SubwayStationETA` API endpoint, but it looks like it is broken in the app.  
With further inspection of the code, I found a URL linking to a SQLite database, which all bus stations, their coordinates and names are saved.  
```java
package com.irantracking.tehranbus.common.api;
  
import kotlin.Metadata;
import org.jetbrains.annotations.NotNull;
import retrofit2.Call;
import retrofit2.http.GET;
  
public interface StringApi {
    @GET("http://www.irantracking.com/ETAAPP/ANDROID")
    @NotNull
    Call<String> getDatabaseAddress();
}
```
Vising the page, it returns a database name  
(e.g. `ECF2405141154.sqlite.gz`)  
and it is appended to URL to obtain a download link.  
(`http://www.irantracking.com/ETAAPP/ANDROID/ECF2405141154.sqlite.gz`).  
Opening the database, it has the following schema:
```sql
CREATE TABLE Stations
  (
     StationCode INTEGER PRIMARY KEY,
     StationName TEXT,
     Longitude   REAL,
     Latitude    REAL,
     Address     TEXT
  );
CREATE TABLE Routes
  (
     RouteID         INTEGER PRIMARY KEY,
     RouteCode       INTEGER,
     Direction       TEXT,
     RouteType       INTEGER,
     OriginationName TEXT,
     DestinationName TEXT,
     Vertices        TEXT
  );
CREATE TABLE RouteStations
  (
     RouteID       INTEGER,
     StationCode   INTEGER,
     StationOrder  INTEGER,
     IsLastStation INTEGER,
     PRIMARY KEY (RouteID, StationCode),
     FOREIGN KEY(RouteID) REFERENCES Routes(RouteID),
     FOREIGN KEY(StationCode) REFERENCES Stations(StationCode)
  );
CREATE TABLE LastUpdate
  (
     LastUpdateLong INTEGER PRIMARY KEY,
     LastUpdateDate TEXT
  ); 
```
A note about `Vertices`, it is just an array of (lat, long), flattened.  
### Subway API
The current bus ETA API works well, but I wanted to have subway eta as well, so I decided to focus on that.   
The `SubwayEtaRequest` class is basically a data class with one attribute, `StationID`.   
But after a lot of searching, I found the reason the app couldn't show subway station eta.  
Unlike the bus ETA API, the subway ETA API uses the following API:
```java
public interface SubwayRouteStationApi {
  @POST("SubwayRoutesStations")
  @NotNull
  Single<Response<SubwayRoutesStationsResponse>> subwayRoutesStations();
}
```
`SubwayRoutesStationsResponse` is just a data class containing the station names, codes and routes.  
To inspect the response from the server, I wrote this simple Python script:
```python
import requests
url = "https://application2.irantracking.com/modsapi/api/PublicTransport/SubwayRoutesStations"
response = requests.post(url, headers={"User-Agent":"okhttp3/10.0"})
print(response.text)
``` 
The server returns:
```json
{"ServerTime": 1724921799615, "Routes": [], "Stations": [], "RouteStations": []}
```
Yeah, the server returns nothing. And that's why it is not working. To further confirm this, I used [PCAPdroid](https://emanuele-f.github.io/PCAPdroid/), and that is also the case (I may write another blog post on this).  
But maybe we could brute force it?
### Brute-forcing the API
OK, Let's be honest, **brute-forcing public free APIs is NOT OK**, but this API is not even used in the app, so I don't think anyone would be bothered, and I am not going to abuse this API and flooding it with requests, just around 500 requests for once, and then it is done.  
Tehran has around 160 stations currently, with more being built every year (hopefully). We have to start somewhere, so I tested this Python script:
```python
import requests
url = "https://application2.irantracking.com/modsapi/api/PublicTransport/SubwayStationETA"
data = {"StationID": 0}
print(requests.post(url, json=data, headers={"User-Agent":"okhttp3/10.0"}).text)
```
Which returns: 
```bash
"اشکال در ساختار دیتای ورودی"
```
It translates to "Error input structure". But maybe station IDs don't start from 0? I tried `{"StationID": 1}`.
```json
[
    {
        "ID": 7,
        "StationID": 1,
        "RouteID": 3,
        "RouteCode": null,
        "OriginationName": "تجریش",
        "DestinationName": "کهریزک",
        "StationOrder": 7,
        "StationName": "شهید حقانی",
        "Details": [
            {
                "ETA": "4 دقیقه",
                "ETAValue": 4,
                "ETAValueText": "4"
            }
        ]
    },
    {
        "ID": 37,
        "StationID": 1,
        "RouteID": 103,
        "RouteCode": null,
        "OriginationName": "کهریزک",
        "DestinationName": "تجریش",
        "StationOrder": 24,
        "StationName": "شهید حقانی",
        "Details": [
            {
                "ETA": "1 دقیقه",
                "ETAValue": 1,
                "ETAValueText": "1"
            }
        ]
    }
]
```
This looks like a response we could use :)
Each station returns two responses, because one is from `"کهریزک"` to `"تجریش"` and the other in reverse. 
`"RouteCode"` is still `null`, but it's at least progress. I tried the first 10, they also do work.  
I wrote some Python to check the first 500 IDs and found these station IDs are valid (with their names as the values).
```json
{
  "شهید حقانی": 1,
  "شهید همت": 2,
  // -- snip --
  "قائم": 143,
  "پايانه ۴و۶ فرودگاه مهرآباد": 149,
  "مهدیه": 150
}
```
It seems they are all sequential, But it is missing some stations, and it is also old!  
Tehran's subway has 7 active lanes. It seems to be missing the newer 6 and 7 lanes. Also, the ETA doesn't work for lane 4. (Returns "---" for the eta value.)  
Maybe that is why they haven't enabled this feature in their app.  
### Android App for subway ETA
I wrote a simple android app using Kotlin and [Jetpack Compose](https://developer.android.com/compose), to check the accuracy of the ETA while I'm traveling.  
It is very basic and has hard-coded translations for Persian, it also doesn't have an icon, but hey it gets the job done.  
Fortunately the app does work and the ETAs are accurate to around 1 minute, which is enough for planning things.
  
<figure>
  <img alt="Android App" src="/public/images/api-extraction-from-tehran-bus-app/Screenshot_android_app.png" style="max-height: 700px !important;"/>
  <figcaption>Android App showing the ETA for ولی عصر station</figcaption>
</figure>
  
