# eKreta-WebAPI-documentation

The information present in this documentation was aquired by inspecting webpage network activity using Firefox's developer tools (F12 > Network tab)

If you are looking for the MobileAPI documentation, check out [Boapps' lovely doc](https://github.com/boapps/e-kreta-api-docs) on it.

## School ID
First of all, the school ID should be obtained from the schools' eKérta address:
```
https://klikXXXXXXXXX.e-kreta.hu/Adminisztracio/Login
```
Where the first portion of the address (klikXXXXXXXXX) is what we need for fursther requests.

## Logging in (getting auth cookie)
#### Config for the "client":
```C#
CookieContainer cookies = new CookieContainer();
HttpClientHandler handler = new HttpClientHandler();
handler.CookieContainer = cookies;
HttpClient client = new HttpClient(handler);
```

#### Credential check:
```C#
string json = ("{\"UserName\":\"" + username + "\",\"Password\":\"" + password + "\"}");
var content = new StringContent(json, Encoding.UTF8, "application/json");
HttpResponseMessage response = client.PostAsync("https://klikXXXXXXXX.e-kreta.hu/Adminisztracio/Login/LoginCheck", content).Result;
contents = await response.Content.ReadAsStringAsync();
```
* This is necessary to obtain an auth cookie.
```json
{"ErrorCode":"Ok","ErrorMessage":"Sikeres bejelentkezés.","Success":true,"WarningMessage":""}
```

#### Get and store cookie:
```C#
string cookie_to_use = "";
Uri uri = new Uri("https://klikXXXXXXXXX.e-kreta.hu/");
IEnumerable<Cookie> responseCookies = cookies.GetCookies(uri).Cast<Cookie>();
foreach (Cookie cookie in responseCookies)
{
  cookie_to_use += (cookie.Name + ": " + cookie.Value);
}
```
* Alternatively it can be parsed out directly of "response" (in the Credential check part), but this is easier.

## Main page auth
```C#
client.DefaultRequestHeaders.Add("Cookie", cookie_to_use); // use the cookie we just received to authenticate
response = client1.GetAsync("https://klikXXXXXXXXX.e-kreta.hu/Adminisztracio/SzerepkorValaszto").Result;
contents = await response.Content.ReadAsStringAsync();
cookie_to_use = ""; // clean the container
foreach (Cookie cookie in responseCookies)
{
  cookie_to_use += (cookie.Name + ": " + cookie.Value);
}
```
* Add the cookie to the request header to prove auth.
* For whatever reason, the auth cookie received at the start will not work for any other pages except this. or at least I couldn't make the thing accept it anywhere else. I think it is because the WebAPI refreshes the auth cookie at every new page loaded, unlike the mobile version which refreshes it at set intervalls (30 seconds).
* Note: i actually have no clue what "SzerepkorValaszto" does, so I assume it just exists. It's better for everyone if you do too.

## Getting the timetable
```C#
client.DefaultRequestHeaders.Add("Cookie", cookie_to_use);
string URL = ("https://klikXXXXXXXXX.e-kreta.hu/api/CalendarApi/GetTanuloOrarend?tanarId=-1&osztalyCsoportId=-1&tanuloId=-1&teremId=-1&kellCsengetesiRendMegjelenites=false&csakOrarendiOra=false&kellTanoranKivuliFoglalkozasok=false&kellTevekenysegek=false&kellTanevRendje=true&szuresTanevRendjeAlapjan=false")
string dateStart = ("&start=" + DateTime.Now.ToString("yyyy").ToString() + "-" + DateTime.Now.ToString("MM").ToString() + "-" + DateTime.Now.ToString("dd").ToString());
string dateEnd = ("&end=" + DateTime.Now.ToString("yyyy").ToString() + "-" + DateTime.Now.ToString("MM").ToString() + "-" + (Int32.Parse(DateTime.Now.ToString("dd").ToString())+1).ToString());
response = client1.GetAsync(URL + dateStart + dateEnd).Result;
contents = await response.Content.ReadAsStringAsync();
```
* In this instance, the response will contain lessons for only the current day.
* dateEnd is parsed to int with a +1 at the end because of the way eKréta handles days. Lessons from December 14th to December 14th are nonexistent, because the day starts at 00:00am and ends at 00:01am. (Lovely)

#### Server response:
```json
[
  {
    "TantargyKategoria":"XXXXX",
    "allDay":false,
    "borderColor":null,
    "borderStyle":null,
    "color":"#397AC1",
    "colorEnum":12,
    "colorRightLine":null,
    "CsengetesiRendId":XXXXX,
    "datum":"0000-00-00T00:00:00",
    "DisplayTime":false,
    "end":"0000-00-0,T00:00:00",
    "hanyadikora":0,
    "helyettesitesId":null,
    "helyettesitoId":null,
    "hetirend":"XXXXX",
    "isElmaradt":false,
    "maxNapiOraszam":X,
    "OraErvenyessegKezdete":null,
    "OraErvenyessegVege":null,
    "OraKezdete":null,
    "orarendViewEnd":null,
    "orarendViewStart":null,
    "oraszam":"XXXXX",
    "oraType":X,
    "OraVege":null,
    "start":"0000-00-00T00:00:00",
    "Tantargy":"XXXXX",
    "Tema":"",
    "text":null,
    "textColor":"#000",
    "textLineThrough":false,
    "title":"XXXXX\nXXXXX\nXXXXX"
  },
  ...
]
```
* The actual response is not as neatly formatted as it is here. I am sorry.

#### Parsing tips:
* don't use char count as landmarks. Because it is a json, it's the safest to use char: " as a good landmark for parsing.
Example for a simple parser, written to get the first class' starting time (starts after the 65th " char):
```C#
firstclasstart = "";
for (int i = 0; i <= contents.Length; i++)
{              
  if (i2 == 65)
  {
    firstclasstart += contents[i];
    if (contents[i + 1] == '"')
    {
      //do stuff with the info you just got, 
      break;
    }
  }
  if (contents[i] == '"')
  {
    i2++;
  }
}
```
