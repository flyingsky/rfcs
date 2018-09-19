* Start Date: 2018-09-19
* RFC PR: (leave this empty)
* Fusion Issue: (leave this empty)

## Summary

Fusion plugin `fusion-plugin-geoip` is a server only plugin which can get user geolocation information based on user ip. To use this plugin you have to prepare your [mmdb](https://www.maxmind.com/en/open-source-data-and-api-for-ip-geolocation) first.

## Basic example

### fusion-plugin-geoip

1. Register plugin in your `app.js`.

```js
import GeoIpPlugin, {GeoIpToken, GeoIpConfigToken} from 'fusion-plugin-geoip';

// Set config with your mmdb file and defaultValue.
// The defaultValue is returned if cannot find matched record
// If defaultValue is not set, fusion-plugin-geoip will use its own defaultValue which is same to below defaultValue.
app.register(GeoIpConfigToken, {
  dbPath: 'geoip/GeoIP2-City.mmdb',
  defaultValue: {
    continentCode: 'NA',
    continentName: 'North America',
    countryCode: 'US',
    countryName: 'United States',
    cityName: 'San Francisco',
    lat: 37.7758,
    lng: -122.4128,
    timeZone: 'America/Los_Angeles'
  },
});

app.register(GeoIpToken, GeoIpPlugin);
```

2. Use it in your plugin.

```js
createPlugin({
  deps: {geoip: GeoIpToken},
  provides(geoip) {
    return {
      getIp: async (ctx: Context) {
        // It throws error if mmdb file is not loaded successfully.
        // The result will be the defaultValue if we cannot find matched record in mmdb,
        // because it's easy for developer to write code.
        // If no matched record in mmdb file, result.original is undefined.
        const result = await geoip.lookup(ctx);
        console.log(result);
      }
    };
  }
});
```

## Motivation

It's good to auto find the right city based on user ip and display corresponding content for that city. To auto detect geo location based on ip we need [Maxmind database file](https://www.maxmind.com/en/open-source-data-and-api-for-ip-geolocation). Plugin `fusion-plugin-geoip` can get ip from fusion Context object and return geo location information.
	
## Detailed design

### fusion-plugin-geoip

1. `GeoIpConfigToken` defines the `.mmdb` file path and default value if cannot find matched record from mmdb file.
2. `GeoIpPlugin` depends on `GeoIpConfigToken` to find `.mmdb` file. We use pure js module [maxmind](https://www.npmjs.com/package/maxmind) to load and parse `.mmdb` file, which provides the best performance. 
3. `GeoIpPlugin` provides service with async `lookup` function, which accept ip string or fusion `Context`. If it's `Context`, [request-ip](https://www.npmjs.com/package/request-ip) is used to extract ip. Function `lookup` always return `GeoIpResultType` to make client coding easier. If cannot find matched ip, `GeoIpConfigType.defaultValue` is returned. If `defaultValue` is not set, plugin's defaultValue will be returned. If matched ip was not found from mmdb, `GeoIpResultType.original` is `undefined` or `null`.

```js
type GeoIpConfigType = {
  dbPath: string,
  defaultValue?: GeoIpResultType,
};

type GeoIpResultType = {
  continentCode: string,
  continentName: string,
  countryCode: string,
  countryName: string,
  cityName: string,
  lat: number,
  lng: number,
  timeZone: string,
  // Original record from mmdb file. It's undefined if cannot find matched record from mmdb file.
  original?: Object,
};

const defaultGeoIpResult = {
  continentCode: 'NA',
  continentName: 'North America',
  countryCode: 'US',
  countryName: 'United States',
  cityName: 'San Francisco',
  lat: 37.7758,
  lng: -122.4128,
  timeZone: 'America/Los_Angeles',
};
```

## Drawbacks
The service function `lookup` is async function, even `maxmind` module provids sync api to lookup, because we need load `mmdb` in async way to unblock server starting.

## Alternatives
Load `mmdb` in sync way to make `lookup` function as sync. But this will block the server starting. To make server responsive as soon as possible we choose async way to load data but make `lookup` as async.

## Adoption strategy

`mmdb` file should be prepared before using this plugin.

## How we teach this

It's very easy to use.

## Unresolved questions

NA