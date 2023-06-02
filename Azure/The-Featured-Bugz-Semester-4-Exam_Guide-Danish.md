# **The-Featured-Bugz Auction Azure Guide**

## **catalog-service**

Denne service indeholder oprettelse og administration af items i virksomhedens varekatalog. <br>
Den kommunikerer med en tilknyttet MongoDB-database samt identifikation af items, <br>
der skal konverteres til aktive auktioner, hvilket gøres i Auction-servicen.

## **auction-service** 
Denne service er hjertet i auktions-processen, fordi den modtager items som skal laves til en aktiv auktion på unikke id’er. <br>
Hertil er den forbundet med en cache-service der indeholde auktionerne, samt bud fra systemets brugere og brugerens unikke id. <br>
Det er ikke muligt at placere et bud, hvis du byder for lavt eller ikke har modtaget en JWT token. <br>
Hvis et bud er godkendt sørger rabbitMQ for at sende det til Bid-servicen med brugerens ID som hentes fra JWT-token. 

## **authentication-service**
Denne service er ansvarlig for at generere JWT-tokens til autentifikation af brugere. <br>
Når brugeren logger ind med brugernavn og adgangskode, som valideres af user-servicen, vil de modtage en JWT-token. <br>
Det er nødvendigt med denne token for at udføre autoriserede HTTP-kald i systemet. 

## **user-service**
Denne service styrer brugeradministrationen og er forbundet til sin egen MongoDB-database, <br>
der indeholder brugerdata. Når en bruger logger på vil de modtage en JWT-token, <br>
som Auction-servicen kan kalde på for at bekræfte de er logget på samt findes brugerens unikke ID. 

## **bid-service**
Denne service håndterer auktionshistorikken ved at modtage data via RabbitMQ. <br>
Data repræsenterer budhistorikken fra igangværende samt tidligere auktioner,<br>
som gemmes i en MongoDB-database.

## **Ip adresse: http://13.68.145.202:4000**

## **Trin 1: Opret en item i catalog-service**
For at starte en auktion skal du oprette en item i catalog-service. <br>
Lav en HTTP **`POST`**-anmodning til endepunktet **`/item/postItem`** med følgende **`JSON-data`**:

```json
{
  "ItemID": 1,
  "ItemName": "Stol",
  "ItemDescription": "meget fin stol",
  "ItemStartPrice": 100,
  "ItemCurrentBid": -1,
  "ItemSellerID": 1,
  "ItemStartDate": "2023-05-25T00:00:00Z",
  "ItemEndDate": "2023-05-30T00:00:00Z"
}
```
Dette vil tilføje varen til MongoDB-databasen. 
## **Trin 2: Opret auktioner fra catalog-service**
Efter at have oprettet en vare i catalog-service kan du gå videre med at oprette auktioner fra kataloget. <br>
Dette kræver en **`JSON-array af items`**, der indeholder en eller flere item. I dette tilfælde har vi kun tilføjet en enkelt item,
som blev oprettet i trin 1. Foretag en HTTP **`POST`**-anmodning til slutpunktet **`/item/postItemsToAuction`** med følgende **`JSON-data`**:
```json
[
  {
    "ItemID": 1,
    "ItemStartPrice": 100,
    "ItemEndDate": "2023-05-25T00:00:00Z"
  }
]
```
Dette vil oprette auktionen for de items og tilføje den til cache(redis-servicen) ved hjælp af auktion-servicen
## **Trin 3: Opret en bruger i User-service**

For at placere et bud skal du først oprette en bruger i User-service. <br>
Foretag en HTTP **`POST`**-anmodning til slutpunktet **`/user/postuser`** med følgende **`JSON-data`**:
```json
{
  "UserID": 1,
  "UserName": "John",
  "UserPassword": "password123",
  "UserEmail": "john@example.com",
  "UserAddress": "123 Main Street"
}
```
Dette vil oprette en bruger med de angivne oplysninger og gemme dem i User-service's MongoDB-database.

## **Trin 4: Modtag en JWT-token fra authentication-service**
For at kunne placere et bud skal du have en JWT-token. For at opnå dette skal du logge ind og modtage en token. <br>
Foretag en HTTP **`POST`**-anmodning til slutpunktet **`/auth/login`** med følgende **`JSON-data`**, der indeholder brugernavn og adgangskode:
```json
{
  "UserName": "John",
  "UserPassword": "password123"
}
```
Dette vil returnere en JSON-respons, der indeholder brugeroplysninger og en token:
```json
{
  "userID": 1,
  "userName": "John",
  "userPassword": "password123",
  "userEmail": "john@example.com",
  "userAddress": "123 Main Street",
  "token": "<JWT TOKEN>"
}
```
## **Trin 5: Placer et bud i auction-service**
Med JWT-tokenet, som blev modtaget i trin 4, kan du nu placere et bud. <br>
Foretag en HTTP **`POST`**-anmodning til slutpunktet **`/auction/PostAuctionBid`** med følgende **`JSON-data`**:
```json
{
  "bidID": 1,
  "auctionID": 1,
  "bidPrice": 150,
  "bidUserID": 2
}
```
Udskift værdierne i henhold til dine budoplysninger. Dette vil indsende budet til Auction-service, <br>
og hvis budet godkendes, vil det blive sendt til Bid-service via RabbitMQ, hvor Bid-service workeren sender det ned til en mongodb database.