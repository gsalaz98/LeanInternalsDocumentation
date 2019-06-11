### Conclusion
Author: Gerardo Salazar

> ### [Section 2.00 - Conclusion](#2.00)

I've concluded that it is reasonable to add a new field to the `SubscriptionDataConfig` indicating that we should use 
map files to resolve certain pieces of custom data with ticker-based source names. Whether implementing a new method or retrofitting
the existing methods to accept a boolean switch to use map files for custom data is up for debate.

Nonetheless, some work will have to be done in order to allow the acceptance of `BaseData` instances to use map files. There were instances where the
map file loading would only occur if the `SecurityType` equaled `Equity` or `Options`, which happens to not be the case.

> ### [Section 2.01 - Closing Remarks](#2.01)

If you have anything to add or correct to this document, please email me.

Thank you for taking the time to read this. I hope you learned something about the `AddData<T>()` methods,
because I certainly did.
