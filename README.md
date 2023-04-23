# VirTracks

This is the official GitHub repository for [VirTracks](https://virtracks.stelltis.ch/), an alternative 3rd-party EDR for [SimRail](https://store.steampowered.com/app/1422130/SimRail__The_Railway_Simulator/). Currently, there's only this README in the repository, as VirTrack's source code is not, and probably will not, be open sourced.

In the future, however, parts of it could become open, like translations, if it ever gets around to support those.

## Data Sources
VirTracks takes its data from multiple sources.

### SimRail APIs

These are APis that are hosted by the SimRail team and act as "official" data sources. They're publicly accessible, but to my knowledge there's no official statement on restrictions on usage, availability or anything else. Some services cache data and have a rate limitation. Please be considerate when using these.

- Timetable data is fetched from [SimRail's AWS API's /getAllTimetables](https://api1.aws.simrail.eu:8082/api/getAllTimetables?serverCode=pl1).
- Timezone data is fetched from [SimRail's AWS API's /getTimezone](https://api1.aws.simrail.eu:8082/api/getTimeZone?serverCode=pl1).
- Server data is fetched from  [SimRail's map API's /servers-open](https://panel.simrail.eu:8084/servers-open).
- Station data is fetched from [SimRail's map API's /stations-open](https://panel.simrail.eu:8084/stations-open?serverCode=pl1)
- Train data is fetched from [SimRail's map API's /trains-open](https://panel.simrail.eu:8084/trains-open?serverCode=pl1)

### Database

VirTracks keeps a database of train events and positions. This allows it to extract useful data, such as signal positions, train delays, or even replays of traffic within a certain timeframe. This database is not public.

# VirTracks API
 
 VirTracks uses a public-ish API to provide the data to the clients. These APIs are subject to change and not meant ofr public consumption, there's no support for it and if you use it, be aware that I might change or just shut them down whenever. But because I can't really stop anyone from doing it, here's a short explanation of what each API does.
 
 All VirTracks API are behind a CloudFlare cache.
 
## Masterdata (REST)

This data is server-independent and does currently not changed. Most of it is fetched on start of the backend or frontend, and not updated.

 - GET `/api/masterdata/servers` is more or less a cached pass-through of the official `/servers-open`. Its main purpose is to prevent calls from VirTracks users to the SimRail servers.
 - GET `/api/masterdata/timezones` is a dictionary of timezones. It allows fetching all timezones for all servers at once, without having to query the official API once per server.
 - GET `/api/masterdata/stations` returns a list of stations, including a VirTracks-internal ID.
 - GET `/api/masterdata/timetable` is similar to the official `/getAllTimetables`, but cached and changed by VirTracks. It uses a VirTrack-internal station id, and adjusts or removes stations that are not of interest to VirTracks. Therefore, it does not represent the "full" timetable as provided by the official API.
 
## Livedata (REST)
 - GET `/api/servers/{serverCode}/state` returns the server state. This call is almost always stale and returns a train and list of events that happened, as well as a timestamp that can be used to enable real-time streaming using SignalR.
 
## Main (SignalR)
 
This SignalR endpoint is available under `/api/main` and is responsible for all data streaming to the clients.

### Server-Methods

These methods can be invoked by the client on the server:

- `void SubscribeToServer(string serverCode, string notBeforeString)`: Subscribes to the general stream of events, excluding map updates. In order to use this method properly, GET the state of a server first, then pass the `modifiedAt` into the call to `SubscribeToServer`. If a server was already subscribed, the old subscription is terminated first.
- `void EnableMapMode(string serverCode)`: Enables streaming of position and velocity data. `SubscribeToServer` must be called before.
- `void DisableMapMode(string serverCode)`: Stops streaming of map data.
- `void Unsubscribe()`: Stops streaming of all data.

### Client-Methods
The following methods will be invoked on a client by the server. All properties are usually abbreviated with no system whatsoever to save bandwidth.

All methods have a `serverCode` parameter, which serves as race condition identifier in case the subscription was ended or changed while a call was in-transit.

- `void UpdateTrains(string serverCode, TrainUpdate[] updatedTrains, TrainEvent[] trainEvents)`: Calle whenever one or more train update or train event has occurred. Train updates change a property on a train; train events define arrivals/departures or similar. The property names are abbreviated. 
- `void UpdateMap(string serverCode, MapUpdate[] mapUpdates)`: Updates map-relevant data for a train.
- `void AddEvents(string serverCode, TrainEvent[] trainEvents)`: Adds events that have happened.
- `void ServerConnectionLost(string serverCode)`: Fired when VirTracks believes that the connection to the SimRail API was lost, or the SimRail server has likely rebooted.
- `void UpdateDispatchers(string serverCode, MapDispatcherUpdate[] dispatcherUpdates)`: Fired when one or more dispatcher changes occur. This can also mean setting a dispatcher to null.
