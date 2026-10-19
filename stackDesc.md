### links:
#### INPOST
##### wszystkie paczkomaty: https://inpost.pl/sites/default/files/points-pl.json?v=20732
{
  "n": "WRO251M",
  "t": "1",
  "d": "Na działce prywatnej",
  "m": "",
  "q": "",
  "f": "004",
  "c": "Wrocław",
  "g": "wroclaw",
  "e": "Szewczenki",
  "r": "dolnośląskie",
  "o": "51-348",
  "b": "2",
  "h": "24/7",
  "co": "PL",
  "l": { "a": 51.14153, "o": 17.1249 },
  "p": 0,
  "s": 1,
  "x": null
}

##### czujniki konkretnego paczkomatu(nie wszystkie paczkomaty mają czujniki): https://inpost.pl/shipx-point-data/41793/WRO251M/air_index_level
{
  "message": "Data source: api-global-points-internal.easypack24.net",
  "air_index_level": "VERY_GOOD",
  "air_sensors": [
    "PM1:0.88:",
    "PM25:1.38:9.22",
    "PM10:1.67:3.7",
    "PRESSURE:1027.86:",
    "HUMIDITY:53.73:",
    "TEMPERATURE:11.75:"
  ]
}

#### ALLEGRO
##### przykładowe zapytanie, generalnie to wypluwa wszystkie klastry(jest 19 w PL) i z każdego klastra można wyciągnąć po 1000 paczkomatów https://edge.allegro.pl/pickup-points/map/rectangle?withAirQuality=false&filter=allegro-one&countryCode=PL&neLat=55.1809695444721&neLng=32.85607349999999&swLat=48.40247637762012&swLng=5.434198499999985
Endpoint / Adres: https://edge.allegro.pl/pickup-points/map/rectangle
Parametry zapytania (Query Params):
neLat, neLng: Współrzędne rogu północno-wschodniego
swLat, swLng: Współrzędne rogu południowo-zachodniego
filter: np. allegro-one
countryCode: PL
withAirQuality: false / true
Przykładowa odpowiedź (fragment):
{
  "points": [
    {
      "id": "2781688",
      "carrierPointId": "AL086DWR",
      "name": "Allegro One Box - AL086DWR",
      "brandKey": "ALLEGRO_APM",
      "coordinates": { "lat": 51.123379, "lng": 17.243699 },
      "airQuality": null
    }
  ],
  "clusters": [
    { "count": 1657, "coordinates": { "lat": 51.1370, "lng": 17.0338 } }
  ]
}
