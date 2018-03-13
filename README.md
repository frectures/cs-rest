## Typische Stolpersteine beseitigen
```
View / Output

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
OK

Select a template:
ASP.NET 4.5.2 Templates / Empty
Add folders and core references for: [x] Web API
[x] Add unit tests
Microsoft Azure [ ] Host in the cloud
OK
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
OK

Tools / NuGet Package Manager / Package Manager Console
PM> install-package Microsoft.AspNet.WebApi.Client
```
Ohne Fehlerbehandlung:
```csharp
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
```csharp
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
## Eigene Datentypen serialisieren
```
Solution Explorer / Solution 'RestServer' / RestServer / Models (right-click) / Add / New Item...
Installed / Visual C# / Code / Class
Name: Kunde
Add
```
Kunde.cs
```csharp
using System;

namespace RestServer.Models
{
    public class Kunde
    {
        public string Nachname { get; private set; }

        public string Vorname { get; private set; }

        public DateTime Geburt { get; private set; }

        public Kunde(string Nachname, string Vorname, DateTime Geburt)
        {
            this.Nachname = Nachname;
            this.Vorname = Vorname;
            this.Geburt = Geburt.Date;
        }
    }
}
```
JSON.NET kommt technisch auch mit öffentlichen Settern klar, aber in fachlichen Klassen will man das nicht. (Alternativ baut man eine saubere fachliche Klasse ohne Properties und mappt von Hand auf eine DTO-Klasse mit komplett öffentlichen Properties.)

DefaultController.cs
```csharp
        // GET: api/Default
        public IEnumerable<Kunde> Get()
        {
            Kunde bill = new Kunde(Nachname: "Gates", Vorname: "Bill", Geburt: DateTime.Parse("1955-10-28"));
            Kunde linus = new Kunde(Nachname: "Torvalds", Vorname: "Linus", Geburt: DateTime.Parse("1969-12-28"));
            return new Kunde[] { bill, linus };
        }
```
## Eigene Datentypen deserialisieren
```
Solution Explorer / Solution 'RestClient' / RestClient (right-click) / Add / New Item...
Installed / Visual C# / Code / Class
Name: Kunde
Add
```
Kunde.cs
```csharp
using System;

namespace RestClient
{
    public class Kunde
    {
        public string Nachname { get; private set; }

        public string Vorname { get; private set; }

        public Kunde(string Nachname, string Vorname)
        {
            this.Nachname = Nachname;
            this.Vorname = Vorname;
        }
    }
}
```
Das Geburtsdatum wurde absichtlich ausgelassen. JSON.NET kommt damit problemlos klar. Man kann sich also auf die Properties beschränken, die einen gerade interessieren. (Alternativ kann man natürlich auch vollständige DTOs verwenden und manuell mappen.)

Program.cs:
```csharp
            IEnumerable<Kunde> deserialized = response.Content.ReadAsAsync<IEnumerable<Kunde>>().Result;

            foreach (Kunde kunde in deserialized)
            {
                Console.WriteLine($"{kunde.Nachname}, {kunde.Vorname}");
            }
```
## Testen mit dem Microsoft Unit Testing Framework

In dem generierten REST-Server ist aufgrund der Option `[x] Add unit tests` bereits ein Test-Projekt enthalten.
Ansonsten muss man das Test-Projekt von Hand anlegen und verdrahten:
```
Solution Explorer / Solution 'RestServer' (right-click) / Add / New Project...
Installed / Visual C# / Test / Unit Test Project
Name: RestServer.Tests
OK

Solution Explorer / Solution 'RestServer' / RestServer.Tests / References (right-click) / Add Reference...
Projects / Solution / [x] RestServer
OK
```
UnitTest1.cs
```csharp
using System;
using Microsoft.VisualStudio.TestTools.UnitTesting;
using RestServer.Models;

namespace RestServer.Tests
{
    [TestClass]
    public class UnitTest1
    {
        [TestMethod]
        public void TestInitialization()
        {
            Kunde linus = new Kunde(Nachname: "Torvalds", Vorname: "Linus", Geburt: DateTime.Parse("1969-12-28"));
            Assert.AreEqual("Torvalds", linus.Nachname);
            Assert.AreEqual("Linus", linus.Vorname);
            Assert.AreEqual(DateTime.Parse("1969-12-28"), linus.Geburt);
        }
    }
}
```
Tests bauen und ausführen
```
Build / Build Solution (F6)
Test / Windows / Test Explorer
Don't show this again
Run All
```
## Vetragsmodell mit Code Contracts for .NET

In den Projekt-Eigenschaften nachschauen, ob es einen Menüpunkt `Code Contracts` (nicht verwechseln mit `Code Analysis`!) gibt. Falls es keinen solchen Menüpunkt gibt:

1. Visual Studio beenden
2. https://marketplace.visualstudio.com/items?itemName=RiSEResearchinSoftwareEngineering.CodeContractsforNET herunterladen und installieren (Alternative Setups findet man auf https://stackoverflow.com/a/46412917)
3. Visual Studio starten

Anschließend sollte man `[x] Perform Runtime Contract Checking` auf `Full` oder zumindest `Preconditions` stellen. (Nachbedingungen werden in der Praxis aus verschiedenen Gründen selten geprüft. Sie dienen aber als praktische Inspirationsquelle für Testfälle.)

Mathematisches Beispiel für Vorbedingungen und Nachbedingungen:
```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics.Contracts;
using System.Linq;

namespace DesignByContractExample
{
    class Program
    {
        public static List<int> factorization(int x)
        {
            Contract.Requires(x >= 1);

            Contract.Ensures(Contract.Result<List<int>>().Count >= 1);
            Contract.Ensures(Contract.Result<List<int>>().IsSorted());
            Contract.Ensures(Contract.Result<List<int>>().Product() == x);

            var factors = new List<int>();
            var root = Math.Sqrt(x);
            for (int i = 2; i <= root; ++i)
            {
                while (x % i == 0)
                {
                    factors.Add(i);
                    x /= i;
                }
            }
            return factors;
        }

        static void Main(string[] args)
        {
            Console.WriteLine(string.Join(" * ", factorization(42)));
        }
    }

    static class Extensions
    {
        public static int Product(this List<int> factors)
        {
            return factors.Aggregate((a, b) => a * b);
        }

        public static bool IsSorted(this List<int> list)
        {
            return list.Zip(list.Skip(1), (a, b) => Tuple.Create(a, b)).All(t => t.Item1 <= t.Item2);
        }
    }
}
```
Die letzte Nachbedingung schlägt dabei fehl, weil `42 = 2 * 3 * 7` ist, die 7 fehlt aber im Ergebnis.

## Messaging

Im Windows-Startmenü `Features` eingeben und `Windows-Features aktivieren oder deaktivieren` auswählen. Falls das Kästchen vor `Microsoft-Message Queue-Server` (sic) leer ist, dieses einmal anklicken. Daraufhin füllt sich dieses mit einem schwarzen Quadrat (es wird nur der direkte Unterpunkt `Microsoft-Message Queue-Serverkernkomponenten` aktiviert und benötigt).
```
Solution Explorer / Solution 'MySolution' / MyApplication / References (right-click) / Add Reference...
[x] System.Messaging
OK
```
Einfaches Beispiel zum Verschicken von Strings:
```csharp
using System;
using System.Messaging;

namespace MessagingExample
{
    class Program
    {
        static void Main(string[] args)
        {
            SendMessages();
            ReceiveMessages();
        }

        const string PATH = @".\Private$\Schlange";

        static void SendMessages()
        {
            if (!MessageQueue.Exists(PATH))
            {
                MessageQueue.Create(PATH);
            }
            using (var messageQueue = new MessageQueue(PATH))
            {
                messageQueue.Send("one");
                messageQueue.Send("two");
                messageQueue.Send("three");
            }
        }

        static readonly XmlMessageFormatter xmlMessageFormatter = new XmlMessageFormatter(new String[] { "System.String, mscorlib" });

        static void ReceiveMessages()
        {
            using (MessageQueue messageQueue = new MessageQueue(PATH))
            {
                messageQueue.Formatter = xmlMessageFormatter;
                try
                {
                    while (true)
                    {
                        Message message = messageQueue.Receive(TimeSpan.FromSeconds(2));
                        string text = message.Body as string;
                        Console.WriteLine(text);
                    }
                }
                catch (MessageQueueException ex)
                {
                    Console.WriteLine(ex.Message);
                }
            }
        }
    }
}
```
Das Senden und Empfangen funktioniert auch über Prozessgrenzen hinweg, durch das `Private$` in diesem einfachen Beispiel allerdings nur auf ein- und demselben Rechner.

Statt Strings kann man auch beliebige Transfer-Objekte mit öffentlichen Properties versenden und empfangen. Dazu muss man an beiden Stellen einen passenden Formatter einstellen:
```csharp
messageQueue.Formatter = new XmlMessageFormatter(new Type[] { typeof(KundeDTO) });
```
