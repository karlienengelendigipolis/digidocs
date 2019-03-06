# Best practices Digipolis

Doel: Best practices en lessons learned uit de praktijk delen in de organisatie.
Doelgroep: developers van projectteams, business architecten en applicatiearchitecten.
Voeg gerust nieuwe best practices toe middels een pull request.

*In de titels wordt aangegeven welk onderdeel must do/verplicht (V) of should do/optioneel (O) is.*

## Inhoudstabel
<!-- TODO -> Dit kun je genereren met doctoc? PC : generated with doctoc (https://www.npmjs.com/package/doctoc) with option --notitle -->

## Document historiek

Versie       | Auteur                 | Datum      | Opmerkingen
------       | -------                | -----      | ------------
0.1          | Karlien Engelen        | 06/03/2019 | Initial draft
---          | ---                    | ---        | ---
---          | ---                    | ---        | ---
---          | ---                    | ---        | ---

## Gebruik het Splunk Dashboard voor troubleshooting (V)

Dit dashboard stelt de ontwikkelaar in staat om alle HTTP-calls van hun applicatie te onderzoeken.

De belangrijkste metrics van een HTTP-call voor het bepalen van de gezondheid van een applicatie zijn de responstijden en de 50X-statuscodes. Het falen van een service gaat vaak gepaard met een 50X response code. De code geeft meer context over het falen.

Zie uitleg: https://wiki.digipolis.be/w/index.php/Splunk_(Ontwikkelaars)

## Stel liveness en readiness probes in (V)

Op alle toepassingen die in openshift draaien, kunnen liveness en readiness probes gedefinieerd worden.

Een liveness probe gaat na of de container nog actief is. Als de liveness probe faalt dan wordt de pod/container herstart.
Een readiness probe controleert of de app/service in container requests kan verwerken. Faalt de readiness probe dan worden er geen requests meer naar deze pod/container gestuurd.

Zie uitleg: https://bitbucket.antwerpen.be/projects/PLAT/repos/documentation/browse/Docker.md#probes

## Kies bewust wat er wel en niet gelogd wordt (V)

Als er gebruik gemaakt wordt van standaard loginstellingen, wordt zowat alles gelogd. Het is beter om werk te steken in het granulair kunnen aansturen van logging niveaus. Doe je dit niet, dan mis je de flexibiliteit om eigen logs op niveau “Information” te loggen en andere logs vanaf “Error”. Dit bemoeilijkt het doorzoeken van logs op het moment dat er een probleem optreedt. 

Geef ook zeker correlationId’s mee in je logging, dit vergemakkelijkt troubleshooting over diverse flows heen.

Uitleg over hoe je het best de loglevels instelt, incl verwijzingen naar voorbeeldcode, vind je hier: https://docs.google.com/document/d/1tnK8JuM8RoK_3JlAKHaCvyySY4a6zNdhZcs9IIOcfQs/edit 
*To do: de google doc omzetten in een github document?*

## Implementatie van health checks (V)

Health checks zorgen ervoor dat het projectteam of P&I kan ingrijpen als er iets met de applicatie of service aan de hand is. Alerting gebeurt via een dashboard en via e-mailnotificaties. 

Momenteel wordt er voor de dashboarding en alerting gebruik gemaakt van Check_Mk en de nieuwe status API.

Voor de implementatie van de health checks is er een nieuwe standaard in de maak die beter aansluit op het gebruik van liveness en readiness probes. De (work in progress) informatie vind je hier: https://github.com/digipolisantwerpdocumentation/status-monitoring 

### Enkele tips bij het implementeren van health checks

-   Zet geen beveiliging op deze endpoints (moeten zeer snel uitgevoerd kunnen worden)

-   Bedenk bij het opstellen van de monitoring endpoints goed welke onderliggende componenten echt kritisch zijn voor het functioneren van je applicatie en welke niet (bijvoorbeeld: de logging engine is niet kritisch, maar de database wel). En bepaal op basis daarvan of je monitoring endpoint op WARN of CRIT komt te staan. 

-   Vraag in de monitoring call de status van onderliggende services op via de status API (het monitoring systeem) in plaats van zelf nog een extra call te doen naar de betreffende service. 

-   Vraag in de monitoring call enkel ping-endpoints op van onderliggende services. Het bevragen van monitoring endpoints heeft geen zin en vertraagt je eigen monitoring calls aanzienlijk. Dit kan ook circular calls introduceren die alles vertragen en uiteindelijk timeouts geven. 

## Upgrade naar de laatste versie van gebruikte technologieeën (V)

Als er van technologieën een nieuwe versie wordt uitgebracht, bevat deze vaak verbeteringen in het kader van security, performantie, bugfixes etc. Het is daarom aangeraden zo snel mogelijk (na voldoende te testen) te upgraden naar de laatste versies. 
Zolang de frameworks nog van security updates worden voorzien, is het geen absolute must om te upgraden. 

Voor ASP .NET Core applicaties is het bijvoorbeeld heel relevant om zo snel mogelijk te upgraden naar de laatste versies van .net core (momenteel 2.2).

## Overweeg het gebruik van toolboxen in applicaties (V)

Er zijn diverse toolboxen beschikbaar via https://github.com/digipolisantwerp 
Het gebruik van toolboxen heeft voor- en nadelen. Het voordeel is dat er voor de ontwikkeling van een applicatie minder tijd moet gestoken worden in het aanmaken van code die regelmatig terugkeert (boilerplate code). 
Het nadeel is dat als een toolbox outdated is of een fout bevat, deze zich ook bevindt in elke applicatie die daarvan gebruik maakt. Ook is het nadeel dat het customizen van code binnen een toolbox vaak moeilijk is. 
Daarom raden we aan om bij het gebruik van een toolbox grondig na te denken over hoe deze zo nuttig mogelijk aan te wenden. Wat eventueel kan helpen is om de toolboxen te ‘forken’, zelf kleine aanpassingen te maken indien nodig en de verbeteringen vervolgens terug te geven aan de community.
*Todo: review doen van de bestaande toolboxen + toolboxen aanbieden in nodejs (nu zijn deze vooral in .net beschikbaar)*

## Analyseer thread starvation (V)

Je wil voorkomen dat er code is die mogelijks [thread starvation](https://en.wikipedia.org/wiki/Starvation_(computer_science)) kan veroorzaken. Er zijn tools om dit voor jouw applicatie te onderzoeken. 

Om .NET code te analyseren op die mogelijke thread starvation kan gebruik gemaakt worden van een handige nuget package: https://github.com/benaadams/Ben.BlockingDetector Voer dan alle calls uit op een lokale machine en analyseer de output op mogelijke verbeteringen. Pas de verbeteringen toe. 
Met deze analyse is bij sommige projecten bijvoorbeeld aan het licht gekomen dat het loggen van events nog blocking werd uitgevoerd.

Een bron voor diagnose van thread starvation in PRD: https://blogs.msdn.microsoft.com/vancem/2018/10/16/diagnosing-net-core-threadpool-starvation-with-perfview-why-my-service-is-not-saturating-all-cores-or-seems-to-stall/ 

## Maak gebruik van HTTP client factory (V)

De lifecycle van een httpclient is vrij complex. Je kan een httpClient als singleton gebruiken zodat je deze niet telkens hoeft op te bouwen, MAAR dit wil ook zeggen dat bij DNS-wijziging van de achterliggende service, de HTTP-client dit niet zal detecteren en zal blijven falen tot de service is herstart. Het telkens opbouwen en afbreken van HttpClient is ook nefast aangezien het disposen van HTTPClients kan betekenen dat de onderliggende connectie gedurende een aantal seconden nog wordt vastgehouden voor deze wordt vrijgegegeven om opnieuw te gebruiken. De betere en standaard manier om HTTPClient te configureren en te gebruiken is via de HttpClientFactory die vanaf ASP .NET Core versie 2.1 beschikbaar is.
https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests 
Leuk hierbij is dat je typed clients kan maken die gevuld kunnen worden met hun eigen HttpMessageHandlers en configuratie en die eenvoudigweg geïnjecteerd kunnen worden. De lifecycle van de agents waar de HttpClient geïnjecteerd wordt is dan altijd transient! 
Voorbeeld: https://bitbucket.antwerpen.be/projects/GDP/repos/bijlage_api_aspnetcore/browse/src/Digipolis.ABS.Bijlage.API/DependencyRegistration.cs 
*To do: deze best practice meer to-the-point beschrijven*

## Zorg voor continue execution door headers read (V)

In .NET kun je kiezen om te wachten tot een volledige response van een HTTP-call is uitgelezen of je kan kiezen om als de headers (die eerst binnenkomen) gelezen zijn, reeds verder te gaan in de executie van de code. Dit versnelt de code aanzienlijk in het geval de content van een response niet uitgelezen dient te worden bij bijvoorbeeld een foutieve statuscode (die in een header zit).

## Code quality control (V)

Software quality control omvat een geheel aan methodieken om de kwaliteit van de code te bewaken. 
Via het Smartops lab wordt bekeken hoe we de code quality control kunnen automatiseren en standaard meenemen in het ontwikkelproces. Intussen is het aangewezen om hiervoor in je projectteam aandacht voor te hebben, zowel bij eigen ontwikkeling als ontwikkeling door de leverancier van een challenge. 
Zorg voor code reviews door ontwikkelaars of applicatie-architecten, doe eens aan pair programming, zorg voor code die herbruikbaar is voor andere projecten, etc.

## Controleer de (technische) afspraken uit QG1 en 2 (V)

Controleer tijdens de projectuitvoer samen met de leverancier eens of er voldaan wordt aan de [technische specificaties](https://github.com/digipolisantwerpdocumentation/technische_specificaties) die in de offertevraag zijn meegegeven, en aan de technische afspraken die zijn vastgelegd op het project-trellokaartje in [Quality Gate 2](https://trello.com/b/limU7MkC/quality-gates-20), bijvoorbeeld de architecturale opzet en de Ops criteria zoals zero downtime deployment, monitoring etc.

## Voorzie een HTTP response time mee in de logging (O)

Logging (via middleware) van alle inkomende requests en afgeronde requests (en bijbehorende timings) is geen overbodige luxe in een micro-service-omgeving. Ook het gebruik van een correlationId die doorgesluisd wordt doorheen inner calls naar andere API’s is een must om het verhaal van een call die van een afnemer komt, volledig te analyseren. 

Het is een aanrader voor elke applicatie binnen Digipolis. 

Zie voor een ASP .NET CORE voorbeeld van een project: https://bitbucket.antwerpen.be/projects/GDP/repos/gdp_common_aspnetcore/browse/src/GdpCommon/Logging/RequestLogMiddleware.cs

## Vergroot de performantie bij serialiseren en deserialiseren (UTF8JSON) (O)

Als je applicatie veel gaat serialiseren en deserialiseren is het noodzakelijk dat het beste, het meest performante en meest efficiënte qua memory-consumption wordt gekozen. Hierbij is bijvoorbeeld gekeken naar Utf8Json. Zie: https://github.com/neuecc/Utf8Json

*Serialisatie = het zodanig omzetten van een object dat dit geschikt wordt voor verzending of opslag op een sequentieel medium*

## Gebruik streams ivm memory en performantie (O)

Door gebruik te maken van disposable streams bij het opstellen van de request en het uitlezen van de response, garandeer je dat de memory consumption niet dramatisch verhoogt bij vele gelijktijdige requests. Ook de globale performantie is beter.

*In computer science, a stream is a sequence of data elements made available over time. A stream can be thought of as items on a conveyor belt being processed one at a time rather than in large batches.*

## Weeg de keus voor asynchrone werking af (O)

Een asynchrone werking zorgt ervoor dat er vanuit de aanroepende toepassing niet gewacht moet worden tot de uitvoering van de call klaar is (wat kan leiden tot een slechte user experience en mogelijk zelfs een time-out in de applicatie) maar dat de applicatie op de hoogte wordt gebracht zodra het antwoord gereed is. Voor onvermijdelijk langlopende API calls kan je best een asynchrone API aanbieden, door ontsluiting via events of door het gebruik van het HTTP 202 patroon. 

Bij het ontwikkelen van API’s is het belangrijk om de interne logica hiervan async op te stellen (ivm thread starvation en throughput). Zo kan je meer requests per seconde aan door slapende threads te hergebruiken voor nieuwe binnenkomende requests. 

Bij het aanspreken van een API vanuit je applicatie maak je best een bewuste afweging om wel of niet een (a)synchrone request te sturen en de user interface wel of niet te blokkeren tijdens de verwerking. Als er een kans is dat de verwerking van de request niet altijd onmiddellijk zal worden voltooid, is de asynchrone werking aangeraden. Maar deze brengt wel een extra complexiteit mee in de applicatie.

## Bouw retry's in bij het aanspreken van onderliggende componenten (O)

Maakt je applicatie gebruik van andere componenten, zorg dan indien mogelijk dat er retry’s zijn ingebouwd. Als je in je applicatie een 502 of 503 terugkrijgt, zou je best automatisch een retry van de call doen.

*To do: de uitwerking hiervan verder bekijken: Vragen we alle applicaties om dit zelf in te bouwen, of kan hiervoor een handreiking gedaan worden zoals een toolbox maken, het op gatewayniveau oplossen, tools aanbieden die dit probleem oplossen en integreren met de bestaande infrastructuur (zoals http://www.thepollyproject.com)?*

## Gebruik de aanbevolen docker images (O)

-   Gebruik zeker geen Alpine (ivm DNS issues), wel Debian
-   Gebruik bij voorkeur Red Hat images (ivm support) - Digipolis heeft daar intern nog weinig ervaring mee, maar zou die ervaring graag opbouwen

Bij vragen of twijfel overleg je met het ALM team. 

## Gebruik het stappenplan voor troubleshooting (O)

Welke stappen kan een developer / projectteam zelf zetten bij probleemanalyse? En wat verwacht de organisatie van een projectteam, voordat er een multidisciplinair team wordt samengesteld om een probleem breder te onderzoeken? 

Zie stappenplan: https://docs.google.com/document/d/1GBcZdBj-nMdlAyn2WYYomvmV2kjwNIkTCtPhwfEofzM/edit?usp=sharing 

Aandachtspunten bij troubleshooting: 
-   Infrastructuurtekening inclusief alle koppelingen 
-   Uittekenen van de flow van calls die het probleem veroorzaakt
-   Reproduceerbaar scenario

# Nuttige links

Informatie over ACPaaS engines, API's en toepassingen vind je op onderstaande plaatsen: 

API store: https://api-store.antwerpen.be/ -> API documentatie van alle Digipolis API's

Github: https://github.com/digipolisantwerp -> technische specificaties, API requirements API design and patterns etc. 

ACPaaS portaal: https://acpaas.digipolis.be -> Businessdocumentatie over ACPaaS componenten

ACPaaS wiki: https://wiki.digipolis.be/ACPAAS/ -> Onderhoudsdocumentatie over ACPaaS componenten

ITvanAtotZ wiki: https://wiki.digipolis.be/itvanatotz -> Functionele en technische documentatie over toepassingen

Bitbucket: Technische documentatie van diverse projecten en [ALM documentatie (over docker, bamboo, deploys etc.)](https://bitbucket.antwerpen.be/projects/PLAT/repos/documentation/browse)


