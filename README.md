## Typische Stolpersteine beseitigen
```
Tools / Options... (ganz unten)

Projects and Solutions / Build and Run
On Run, when projects are out of date: Always build
On Run, when build or deployment errors occur: Do not launch
OK
```
## Ein einfacher REST-Server
```
File / New / Project... (Ctrl+Shift+N)
Installed / Templates / Visual C# / Web
ASP.NET Web Application
Name: RestServer
[ ] Create directory for solution
OK

Select a template:
ASP.NET 4.5.2 Templates / Empty
Add folders and core references for:
[x] Web API
OK

Configure Microsoft Azure Web App
Cancel
```
## Der erste Controller
```
Solution Explorer / Solution 'RestServer' / RestServer / Controllers (right-click) / Add / Controller...
Web API 2 Controller with read/write actions
Add

Controller name: DefaultController
Add

Debug / Start Debugging (F5)
```
Es öffnet sich ein Browser-Fenster auf `localhost:12345` (der tatsächliche Port wird zufällig vergeben) mit der Fehlermeldung `HTTP Error 403.14 - Forbidden` -- das gehört so!
An die URL `/api/Default` anhängen, also `localhost:12345/api/Default`, dann sollte `["value1","value2"]` erscheinen.
Diese Werte kommen aus der `DefaultController.Get()` Methode. Das `IEnumerable` wurde dabei automatisch nach JSON konvertiert.

Wenn man einen anderen Port wünscht, dann kann man diesen wie folgt einstellen:
```
Solution Explorer / Solution 'RestServer' / RestServer (right-click) / Properties / Web / Servers
Project Url: http://localhost:12345/
Create Virtual Directory
OK
```
## Ein einfacher REST-Client
Den Server laufen lassen und **eine zweite Instanz von Visual Studio 2015 starten!**

```
File / New / Project... (Ctrl+Shift+N)
Installed / Templates / Visual C# / Windows
Console Application
Name: RestClient
[ ] Create directory for solution
OK

Tools / NuGet Package Manager / Package Manager Console
PM> install-package Microsoft.AspNet.WebApi.Client
```
Ohne Fehlerbehandlung:
```
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Net.Http.Headers;

namespace RestClient
{
    class Program
    {
        const string server = "http://localhost:12345";
        const string endpoint = "/api/Default";

        static void Main(string[] args)
        {
            HttpClient client = new HttpClient();
            client.BaseAddress = new Uri(server);
            client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));

            HttpResponseMessage response = client.GetAsync(endpoint).Result;

            IEnumerable<string> deserialized = response.Content.ReadAsAsync<IEnumerable<string>>().Result;

            foreach (string s in deserialized)
            {
                Console.WriteLine(s);
            }
        }
    }
}
```
Mit Fehlerbehandlung:
```
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Net.Http.Headers;

namespace RestClient
{
    class Program
    {
        const string server = "http://localhost:12345";
        const string endpoint = "/api/Default";

        static void Main(string[] args)
        {
            HttpClient client = new HttpClient();
            client.BaseAddress = new Uri(server);
            client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
            try
            {
                HttpResponseMessage response = client.GetAsync(endpoint).Result;
                if (response.IsSuccessStatusCode)
                {
                    Console.WriteLine("*** Server hat geantwortet ***");
                    IEnumerable<string> deserialized = response.Content.ReadAsAsync<IEnumerable<string>>().Result;
                    Console.WriteLine("*** Deserialisierung erfolgreich ***");
                    foreach (string s in deserialized)
                    {
                        Console.WriteLine(s);
                    }
                }
                else
                {
                    Console.WriteLine($"Der Server antwortete mit {response.StatusCode}: {response.ReasonPhrase}");
                }
            }
            catch (AggregateException ex) when (ex.InnerException.GetType().ToString().StartsWith("System.Net"))
            {
                Console.WriteLine($"*** {server}{endpoint} nicht erreichbar ***");
                Console.WriteLine(ex.InnerException);
            }
            catch (AggregateException ex) when (ex.InnerException.GetType().ToString().StartsWith("Newtonsoft.Json"))
            {
                Console.WriteLine("*** Deserialisierung fehlgeschlagen ***");
                Console.WriteLine(ex.InnerException);
            }
            catch (Exception ex)
            {
                Console.WriteLine(ex);
            }
        }
    }
}
```
