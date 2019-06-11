## LEAN Custom Data Mapping
Author: Gerardo Salazar

> #### [0.00 - Introduction](#introduction)

Assume the following:

* Ticker `ENRN` gets listed on 1998-01-01T00:00:00 UTC.
* We've requested for SEC custom data for ticker `ENRN` on 1998-01-02T00:00:00 UTC

Consider the following algorithm:

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
*/

using System.Collections.Generic;
using System.Linq;
using QuantConnect.Data;
using QuantConnect.Data.Custom.SEC;

namespace QuantConnect.Algorithm.CSharp
{
    /// <summary>
    /// Demonstration algorithm showing how to use and access SEC data
    /// </summary>
    /// <meta name="tag" content="fundamental" />
    /// <meta name="tag" content="using data" />
    /// <meta name="tag" content="custom data" />
    /// <meta name="tag" content="SEC" />
    public class SECReportDataAlgorithm : QCAlgorithm
    {
        private Symbol _symbol;

        public const string Ticker = "ENRN";

        /// <summary>
        /// Initialise the data and resolution required, as well as the cash and start-end dates for your algorithm. All algorithms must initialized.
        /// </summary>
        public override void Initialize()
        {
            SetStartDate(1998, 1, 2);
            SetEndDate(2019, 5, 31);
            SetCash(100000);

            _symbol = AddData<SECReport10Q>(Ticker).Symbol;
            AddData<SECReport8K>(Ticker);
        }

        public override void OnData(Slice slice)
        {
            var data = slice.Get<ISECReport>();

            foreach (var submission in data.Values)
            {
                Log($"Form Type {submission.Report.FormType}");
                Log($"Filing Date: {submission.Report.FilingDate:yyyy-MM-dd}");

                foreach (var filer in submission.Report.Filers)
                {
                    Log($"Filing company name: {filer.CompanyData.ConformedName}");
                    Log($"Filing company CIK: {filer.CompanyData.Cik}");
                    Log($"Filing company EIN: {filer.CompanyData.IrsNumber}");

                    foreach (var formerCompany in filer.FormerCompanies)
                    {
                        Log($"Former company name of {filer.CompanyData.ConformedName}: {formerCompany.FormerConformedName}");
                        Log($"Date of company name change: {formerCompany.Changed:yyyy-MM-dd}");
                    }
                }

                // SEC documents can come in multiple documents.
                // For multi-document reports, sometimes the document contents after the first document
                // are files that have a binary format, such as JPG and PDF files
                foreach (var document in submission.Report.Documents)
                {
                    Log($"Filename: {document.Filename}");
                    Log($"Document description: {document.Description}");

                    // Print sample of contents contained within the document
                    Log(document.Text.Substring(0, 100));
                    Log("=================");
                }
            }
        }
    }
}
```

> [Section 0.01 - The Problem](#0.01)

Q: What would happen if `ENRN` gets delisted on 2001-12-27T00:00:00 UTC and another company takes over the ticker on 2019-01-01T00:00:00 UTC, which then undergoes a renaming event on 2019-02-01T00:00:00 UTC and the ticker is now named `GRRY`? Additionally, what would happen if on 2019-03-01T00:00:00 UTC, another company takes over the ticker `ENRN`, while the ticker named `GRRY` just renamed?

A: The user will keep receiving updates for the symbol `ENRN` after the symbol has been renamed from ENRN, and not track the rename event of the ticker. The intended behavior is that we would receive updates for `GRRY`, and not `ENRN` after the renaming event. The user would have to re-add `ENRN` in order to get updates for ticker `ENRN` after the ticker had been renamed. In addition, if we wanted to access historical `GRRY` SEC records, we would load data for `GRRY` from the past, even if the ticker is not associated with the current company that uses the `GRRY` ticker.

This is not the intended behavior. We need to figure out a way to include the usage of map files into custom data, somewhere along the `SubscriptionData` stack in LEAN, and update the symbol we read data from and map correctly historically to previous values.

In order to begin, we must first consider whether we want to edit the existing `AddData<T>` methods, or add a new method that can indicate whether or not we want to update the data with mapfiles. Maybe something like `renameEvents = false`?
Because not all data is created with symbols that can change over time (e.g. weather data), we should consider making it an opt-in feature. Sentiment data suffers from the same problem described above since the identifier for the data is based on tickers.

In the next document, we will begin enumerating through the LEAN data stack and hopefully identify a suitable place to begin implementing supporting map files for custom data sources.


