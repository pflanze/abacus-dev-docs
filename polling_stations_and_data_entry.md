The structs handled in the app and the tables don't correspond closely; there is (generally?) an underlying Row struct, though. E.g. `DataEntryRow` vs. `DataEntrySourceContext`.



The table `polling_stations` lists all the polling stations, and is initialized via the example database reset functionality.

The table `data_entries` is basically an "extension record" for `polling_stations`, meaning *when* there was data entry happening, the `data_entries` row adds information "to" the polling station. There is always 0 or 1 such entry linked from `polling_stations.data_entry_id`, but it can also be linked from `polling_stations.prev_data_entry_id`. Why the latter? XXX


`polling_station_repo::link_data_entry`

The only use of this function is in `polling_station_repo::create_data_entry`

    let data_entry = data_entry_repo::create_empty(&mut tx).await?;
    link_data_entry(&mut tx, polling_station_id, data_entry.id).await?;

Also:

    /// Clear `data_entry_id` on a polling station (counterpart to `link_data_entry`).
    pub async fn unlink_data_entry(

which is only called in `data_entry::delete_data_entry_for_polling_station`

    polling_station_repo::unlink_data_entry(conn, polling_station_id).await?;
    let data_entry = data_entry_repo::delete(conn, data_entry_id).await?;

(Why are `data_entry::delete_data_entry_for_polling_station` and `polling_station_repo::create_data_entry` not in the same module?)

Both of these only deal with the `polling_stations.data_entry_id`!

And `polling_stations.prev_data_entry_id`? This is set in `polling_station_repo::update`


`PollingStationRow` vs. `PollingStationForSession`?


backend/src/repository/data_entry_repo.rs
has some comment:

    // Query to find the most recent results for each polling station.
    // For each PS, it checks:
    // 1. If the PS has its own Definitive data entry, use that
    // 2. If there's no investigation requiring corrected results, use the previous data entry
    //    (pointed to by prev_data_entry_id, which already skips intermediate sessions without results)
    // 3. Otherwise, NULL (results are expected but not yet available)


The only place that is setting the `prev_data_entry_id` field is 
`polling_station_repo::duplicate_for_committee_session`. 

Which is only called in `repository::committee_session_repo::create`. (Which we guess explains why the function is called `duplicate_*for_committee_session*`.)

Which is called from `api::committee_session::create_committee_session` 
and `test_data_gen::generators::create_test_election`.

The former of which is called from `api::election::create_election_with_committee_session` (called from `api::election::import_gsb_election` and `api::election::import_csb_election`) 
and `api::committee_session::committee_session_create` (used in `api::committee_session::router` and routed at
`path = "/api/elections/{election_id}/committee_sessions"`
).

(Our temporary data:

    polling_stations

    5211	6		502
    518		5		501
    116		1		106
    115		1		105
    114		1		104   -- that was me
    529		6  501	

    104   firstentryfinalized
    105   Empty
    106   Empty
    501   Definitive
    502   Definitive
    801   Definitive

)

We don't know how to trigger the `"/api/elections/{election_id}/committee_sessions"` router; example of creating a new polling station (are all requests actually logged like this?):

```
2026-06-27T13:59:44.272171Z  INFO request{method=POST uri=/api/data_entries/105/1 version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=20 ms status=200
2026-06-27T14:15:57.553278Z  INFO request{method=GET uri=/api/account version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
2026-06-27T14:15:57.557466Z  INFO abacus::api::middleware::authentication::middleware: Session extended for user 5
2026-06-27T14:15:57.602024Z  INFO request{method=GET uri=/api/elections version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=1 ms status=200
2026-06-27T14:16:03.224200Z  INFO request{method=POST uri=/api/login version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=237 ms status=200
2026-06-27T14:16:03.262929Z  INFO request{method=GET uri=/api/elections version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=1 ms status=200
2026-06-27T14:16:07.526714Z  INFO request{method=GET uri=/api/elections version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
2026-06-27T14:16:12.438632Z  INFO request{method=GET uri=/api/users version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
2026-06-27T14:16:14.812041Z  INFO request{method=GET uri=/api/elections version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=1 ms status=200
2026-06-27T14:16:16.459401Z  INFO request{method=GET uri=/api/elections/1 version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=2 ms status=200
2026-06-27T14:16:16.465414Z  INFO request{method=GET uri=/api/elections/1/status version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=1 ms status=200
2026-06-27T14:16:16.479482Z  INFO request{method=GET uri=/api/elections/1 version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=1 ms status=200
2026-06-27T14:16:25.260590Z  INFO request{method=GET uri=/api/users version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=200
2026-06-27T14:16:25.279596Z  INFO request{method=GET uri=/api/elections/1/status version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=2 ms status=200
2026-06-27T14:16:25.279596Z  INFO request{method=GET uri=/api/elections/1 version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=2 ms status=200
2026-06-27T14:16:38.253922Z  INFO request{method=GET uri=/api/elections/1/polling_stations version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=1 ms status=200
2026-06-27T14:17:23.664283Z  INFO request{method=POST uri=/api/elections/1/polling_stations version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=6 ms status=201
2026-06-27T14:17:23.672387Z  INFO request{method=GET uri=/api/elections/1 version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=2 ms status=200
2026-06-27T14:17:23.687270Z  INFO request{method=GET uri=/api/elections/1/polling_stations version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=1 ms status=200
```

