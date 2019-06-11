## Review of the LEAN Data Stack
Author: Gerardo Salazar

> #### [Section 1.00 - Introduction to Data](#1.00)

We will begin enumerating the entire LEAN data stack from the top-down, starting with the `AddData`, `AddEquity`, Add...` etc. methods.
Then, we will begin enumerating from the bottom of the stack, starting with `BaseData.GetSource()` and `BaseData.Reader()` methods.

For the sake of this document, we will be focusing on custom data as it's simpler and is the data source most users will have to deal with.

We add custom data by calling the following method:

```C#
SetStartDate(1998, 1, 1);
SetEndDate(2019, 5, 31);

AddData<T>(string, [ Resolution = Resolution.Minute])
where 
    T: BaseData 
```
This will load custom data for type `T` that derives `BaseData` from the date specified in `SetStartDate` and ends at `SetEndDate`.

As stated in the previous document, not all BaseData data sources are equal in that the data is mapped to specific tickers. In fact, consider 
economic data, where the value for `Ticker` (i.e. first `AddData<T>()` argument) is a country, or city name for hypothetical NOAA data.

Because of that fact:

1. We must make an opt-in feature for `AddData<T>` that can instruct the LEAN stack to process map files
for custom data that is referenced by ticker names.
2. Consider whether we're going to edit existing methods, or create a new method that accepts an argument or configuration parameters

> #### [Section 1.01 - `AddData<T>` Methods](#1.01)

Let's begin our review by identifying various `AddData<T>` method signatures:

```C#
public Security AddData<T>(string ticker, Resolution resolution = Resolution.Minute)
    where T : IBaseData, new()
{
    //Add this new generic data as a tradeable security:
    // Defaults:extended market hours"      = true because we want events 24 hours,
    //          fillforward                 = false because only want to trigger when there's new custom data.
    //          leverage                    = 1 because no leverage on nonmarket data?
    return AddData<T>(ticker, resolution, fillDataForward: false, leverage: 1m);
}
```

* This is the most commonly used method to add custom data.
* Calls the following method:

```C#
public Security AddData<T>(string ticker, Resolution resolution, bool fillDataForward, decimal leverage = 1.0m)
    where T : IBaseData, new()
{
    return AddData<T>(ticker, resolution, TimeZones.NewYork, fillDataForward, leverage);
}
```

* Calls the following method:

```C#
public Security AddData<T>(string ticker, Resolution resolution, DateTimeZone timeZone, bool fillDataForward = false, decimal leverage = 1.0m)
    where T : IBaseData, new()
{
    //Add this custom symbol to our market hours database
    MarketHoursDatabase.SetEntryAlwaysOpen(Market.USA, ticker, SecurityType.Base, timeZone);

    //Add this to the data-feed subscriptions
    var symbol = new Symbol(SecurityIdentifier.GenerateBase(ticker, Market.USA), ticker);

    //Add this new generic data as a tradeable security:
    var config = SubscriptionManager.SubscriptionDataConfigService.Add(typeof(T),
        symbol,
        resolution,
        fillDataForward,
        extendedMarketHours: true,
        isCustomData: true);
    var security = Securities.CreateSecurity(symbol, config, leverage);

    AddToUserDefinedUniverse(security, new List<SubscriptionDataConfig>{ config });
    return security;
}
```

Let's take a look at the `Subscriptionmanager.SubscriptionDataConfigService`.

`SubscriptionDataConfigService` eventually trickles down into the `Reader` method of `BaseData` as a `SubscriptionDataConfig` object.
It might be worthwhile to indicate to the `Reader` method that we're using map files so that it can load the files from disk.

However, that's a very crude approach and can lead to implementation issues. I believe we should keep the reader as dumb as possible and handle the 
updates higher up the stack. Let's explore other solutions by looking further down the stack.

> #### [Section 1.02 - LEAN Custom Data Stack From The Bottom Up](#1.02)

Assume we have the following class defined as the BaseData implementation for our PsychSignal custom data:

```C#
/*
 * QUANTCONNECT.COM - Democratizing Finance, Empowering Individuals.
 * Lean Algorithmic Trading Engine v2.0. Copyright 2014 QuantConnect Corporation.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 *
*/

using System;
using System.Globalization;
using System.IO;

namespace QuantConnect.Data.Custom.PsychSignal
{
    public class PsychSignalSentimentData : BaseData
    {
        /// <summary>
        /// Bullish intensity as reported by psychsignal
        /// </summary>
        public decimal BullIntensity;

        /// <summary>
        /// Bearish intensity as reported by psychsignal
        /// </summary>
        public decimal BearIntensity;

        /// <summary>
        /// Bullish intensity minus bearish intensity
        /// </summary>
        public decimal BullMinusBear;

        /// <summary>
        /// Total bullish scored messages
        /// </summary>
        public int BullScoredMessages;

        /// <summary>
        /// Total bearish scored messages
        /// </summary>
        public int BearScoredMessages;

        /// <summary>
        /// Bull/Bear message ratio.
        /// </summary>
        /// <remarks>If bearish messages equals zero, then the resulting value equals zero</remarks>
        public decimal BullBearMessageRatio;

        /// <summary>
        /// Total messages scanned.
        /// </summary>
        /// <remarks>
        /// Sometimes, there will be no bull/bear rated messages, but nonetheless had messages scanned.
        /// This field describes the total fields that were scanned in a minute
        /// </remarks>
        public int TotalScoredMessages;

        /// <summary>
        /// Retrieve Psychsignal data from disk and return it to user's custom data subscription
        /// </summary>
        /// <param name="config">Configuration</param>
        /// <param name="date">Date of this source file</param>
        /// <param name="isLiveMode">true if we're in livemode, false for backtesting mode</param>
        /// <returns></returns>
        public override SubscriptionDataSource GetSource(SubscriptionDataConfig config, DateTime date, bool isLiveMode)
        {
            return new SubscriptionDataSource(
                Path.Combine(
                    Globals.DataFolder,
                    "equity",
                    config.Market,
                    "alternative",
                    "psychsignal",
                    $"{config.Symbol.Value.ToLower()}.zip#{date:yyyyMMdd}.csv"
                ),
                SubscriptionTransportMedium.LocalFile,
                FileFormat.Csv
            );
        }

        /// <summary>
        /// Reads a single entry from psychsignal's data source.
        /// </summary>
        /// <param name="config">Subscription data config setup object</param>
        /// <param name="line">Line of the source document</param>
        /// <param name="date">Date of the requested data</param>
        /// <param name="isLiveMode">true if we're in live mode, false for backtesting mode</param>
        /// <returns>
        ///     Instance of the T:BaseData object containing psychsignal specific data
        /// </returns> 
        public override BaseData Reader(SubscriptionDataConfig config, string line, DateTime date, bool isLiveMode)
        {
            try
            {
                var csv = line.Split(',');

                var timestamp = new DateTime(date.Year, date.Month, date.Day).AddMilliseconds(
                    Convert.ToDouble(csv[0])
                );
                var bullIntensity = Convert.ToDecimal(csv[1], CultureInfo.InvariantCulture);
                var bearIntensity = Convert.ToDecimal(csv[2], CultureInfo.InvariantCulture);
                var bullMinusBear = Convert.ToDecimal(csv[3], CultureInfo.InvariantCulture);
                var bullScoredMessages = Convert.ToInt32(csv[4]);
                var bearScoredMessages = Convert.ToInt32(csv[5]);
                var bullBearMessageRatio = Convert.ToDecimal(csv[6], CultureInfo.InvariantCulture);
                var totalScannedMessages = Convert.ToInt32(csv[7]);
                
                return new PsychSignalSentimentData
                {
                    Time = timestamp,
                    Symbol = config.Symbol,
                    BullIntensity = bullIntensity,
                    BearIntensity = bearIntensity,
                    BullMinusBear = bullMinusBear,
                    BullScoredMessages = bullScoredMessages,
                    BearScoredMessages = bearScoredMessages,
                    BullBearMessageRatio = bullBearMessageRatio,
                    TotalScoredMessages = totalScannedMessages
                };
            }
            catch
            {
                return null;
            }
        }
    }
}
```

Let's look at methods that call `GetSource` to see where the beginning of where the `SubscriptionDataSource` is used.

Following the references to the `GetSource` method, we can see that it's called from a lot of places. But in particular, we're focused on way the core parts of
LEAN handle this custom data and how to load it. 

We start by looking at `ResolveDataEnumerator` under project `QuantConnect.Lean.Engine` in file `DataFeeds/SubscriptionDataReader.cs`.

We need to find a suitable place to perhaps alter the `Symbol` used in the config with the prior company ticker.

The configuration is already defined, which is set from the constructor of `SubscriptionDataReader`. There are two instances where an instance is created:

* `SubscriptionDataReaderHistoryProvider.cs` in project `QuantConnect.Lean.Engine`
* `SubscriptionDataReaderSubscriptionEnumeratorFactory.cs` in project `QuantConnect.Lean.Engine`

Both of these instances create an instance of `MapFileResolver`, which gets all map files for a given market. This is a good place to start looking into implementing map files
into the stack, since it is already being used for equities and options.

Let's track the lifespan of `mapFileResolver` inside the `SubscriptionDataReader` instance to see where it leads us:

Note: Assume commit is master 9ee707503

1. Assigned to private member variable on line 149
2. First used on line 229 to resolve map file (important!)
    - Note that the `if` statement above line 229 prohibits custom data from getting a mapFile assigned

Following the lifespan of `mapFile` and `_mapFile` in the `SecurityType.Equity` path:

1. First, `_mapFile` gets assigned empty data, which in turn contains no information on line 222
2. Inside the conditional (line 225+), `mapFile` is assigned to `_mapFileResolver.ResolveMapFile(string ticker, DateTime firstTradeDate)`
3. If `mapFile` contains any entries, the `_mapFile` member variable is set equal to the new resolved map file (i.e. filled with data and not empty).

`_mapFile` is then used inside method `TryGetNextDate()`, which ultimately doesn't matter because the `_mapFile` instance contains no data.

Looking back to method `BaseDataSubscriptionEnumeratorFactory.CreateEnumerator()`, it calls method `GetMappedSymbol()` which loads the mapfile we need
in order to map our current symbol to the right ticker.

The way the `Symbol` is loaded is through the `SubscriptionRequest` configuration, which contains various fields that determines the flow of execution.
Let's find out where exactly it's constructed along the chain of `AddData<T>()`.

Looking at the final `AddData<T>()`, we can see that the configuration is created and added to the `SubscriptionManager` with various fields that returns a `SubscriptionDataConfig` The configuration from defaault `AddData<T>(string Ticker)` resolves to the following configuration:

```C#
new SubscriptionDataConfig() {
    Type = typeof(T),
    SecurityType = SecurityType.Base,
    Symbol = ???,
    TickType = Trade,
    Resolution = Resolution.Minute,
    Increment = ???,
    FillDataForward = true,
    ExtendedMarketHours = false,
    IsInternalFeed = false,
    IsCustomData = true,
    ...
};
```

It then gets passed on to `AddToUserDefinedUniverse`, which adds a user defined universe to the `_pendingUniverseAdditions`. Then, in `QCAlgorithm.Universe.OnEndofTimeStep()`, on line 150, `subscriptionDataConfig` from `_pendingUserDefinedUniverseSecurityAdditions[i].SubscriptionDataConfigs` are added to _pendingUserDefinedUniverseAddition.Universe`, which are then in turn added to the `UniverseManager`.

A comment above where the `UniverseManager` additions happen states the following: `// finally add any pending universes, this will make them available to the data feed`

This means we need to start looking towards the datafeed, most likely referring to the `DataManager` class in `QuantConnect.lean.Engine`. In the constructor, there is an event attached to adding a new `Universe` to the `UniverseManager`. Inside, most importantly, a new `SubscriptionRequest` is passed to method `AddSubscription`, which checks for duplicate configurations, and passes the request to `_dataFeed.CreateSubscription()`. 

Inside the `FileSystemDataFeed` (we assumed this is the path we will most likely take with regards to the datafeed), it will then call `CreateDataSubscription()` with the request.

It then makes sure that there are tradable days in the request, then calls `GetEnumeratorFactory()` with the request. In here, most likely we will default to `SubscriptionDataReaderSubscriptionEnumeratorFactory` because we aren't subscribing to a universe.

Inside the defaulted factory (`SubscriptionDataReader...`), the mapFileResolver is called. We've already covered this part in #[Section 1.02](#1.02). Here, we have access to the 
configuration we created in `AddData<T>()` via `request.Configuration`.

