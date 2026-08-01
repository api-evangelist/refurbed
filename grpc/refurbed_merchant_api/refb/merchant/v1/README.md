# Refurbed Merchant API

The refurbed merchant API is based on [gRPC](https://grpc.io). There is also 
an HTTP version of this API available.

## Rate limits

To keep our systems stable, the API implements rate limiting. So please write a
sensible implementation. For example use batch operations whenever they make
sense.

You can check the state of your rate limits using the following response headers:

- `X-RateLimit-Limit` how many requests you’ve already made in this time window
- `X-RateLimit-Remaining` how many requests you’ve got left in this time window
- `X-RateLimit-Reset` when the time window will reset as a unix timestamp

## Key concepts

Please refer to the fully documented proto files for more details about which
methods and objects are supported in detail. Start with the services to see what
operations are available to you. You can find I/O objects in the services files.
The main data objects are defined in the `resources/` directory.


### HTTP API

If you prefer to use the HTTP version of this API, please refer to the 
OpenAPI definition
(`./refurbed_merchant_api/refb/merchant/v1/api.swagger.json`).
You can use it to automatically generate a client. To display end-user 
documentation from the JSON, you can run the following command and visit 
http://localhost:9999

```
$ docker run -p 9999:8080 \
    -v `(pwd)`/refurbed_merchant_api/refb/merchant/v1:/openapi \
    -e SWAGGER_JSON=/openapi/api.swagger.json \
    swaggerapi/swagger-ui
```

### Data types

Monetary amounts as well as other decimal types which require exact precision
(e.g. exchange rates) are encoded as strings. They use a decimal point as
separator (e.g. `123.45`).

Monetary amounts support a precision of up to 2 decimal digits
(valid: `123`, `123.45`; invalid: `123.456`).

### Orders

Use the `OrderService` to list orders which were released to the merchant (meaning
they are ready to be fulfilled by the merchant). `ListOrders()` offers a variety of
filters to allow you to specify exactly what you want to receive.

You can use the `OrderItemService` to update data on order items, change their state,
or refund them. There is also a refund method in the `OrderService` that acts as a
convenience function to refund all items of an order at once. Both services expose
a dry-run refund method called `CalculateRefundOrder*`.

### Instances

Product variants at refurbed are called instances, they are managed by refurbed.
You can use the `InstanceService` to retrieve individual instances by either their
GTIN or refurbed instance id. This is useful to check if refurbed knows the GTIN
and to find out the exact name set.

### Shipping profiles

Use shipping profiles to define which markets your products can be shipped into,
and the shipping costs which need to be added. Shipping costs can be defined in
any  of the supported currencies returned by the `CurrencyService`.

### Offers

Create offers on individual instances. An `Offer` combines information like
grading, warranty or stock. It also defines a reference price and optional
reference min. price.  These reference prices are the default prices used for
all markets. They are automatically converted into the respective market's
currency, if it doesn't match the offer price's currency. Prices, like shipping
costs, can be defined in any of the supported currencies returned by the
`CurrencyService`.

Prices for the individual markets are represented by the `OfferMarketPrice`
object. By default, they are calculated from the reference prices, but they 
can also be specified explicitly (manual market price management). This is
done via the `MarketOfferService` and the `MarketOffer` objects.

Think of an `Offer` as an entry in our main "offers" table, and of
`OfferMarketPrice` as related market price entries connected to the main
offer in a 1:n relationship.

The `Offer` object always returns all market prices, while a `MarketOffer`
groups the main offer with one single market price.

If you want to create new offers, change their configuration or stock, use
the `OfferService` (market-independent functionality). If you want to set
prices manually for a specific market, or get information about which offers
have/do not have the BuyBox in a given market, use the `MarketOfferService`
(market-specific functionality).


## Generating the client

You can either use the official `protoc` compiler to generate a client for
your programming language of choice, or ask your refurbed contact to send you
an already generated one.

The minimum required `protoc` version is `3.15`. You can find the official
repository [here](https://github.com/protocolbuffers/protobuf).

Example for generating a Python client using a dockerised version of `protoc`:

```
$ docker run \
    -u $(id -u ${USER}):$(id -g ${USER}) \
    -v `pwd`/refurbed_merchant_api:/defs \
    namely/protoc-all \
    -i . \
    -d refb/ \
    -l python
```

This will create a new directory `./refurbed_merchant_api/gen/pb_python` with the
relevant client code inside. You can copy the `refb` directory into your project.


## Using the client

The production system is reachable on host `api.refurbed.com:443`. If needed,
we can also provide a sandbox system for development and testing. You need a
secret token to be able to authenticate with our API. The token has to be set
in the authorization HTTP header/gRPC metadata using method `Plain`
(e.g. `Authorization: Plain f4e133d1-6cc4-4792-be5d-80a446a3d0dd`). Please contact
your account manager to receive your credentials.

Using Python as an example again, you can use the client like in the following
snippet. You can install the dependencies for this example with
`pip install grpcio protobuf google-api-python-client`.


```python
import grpc
from refb.merchant.v1.enums.market_type_pb2 import MarketTypeEnum
from refb.merchant.v1.enums.sort_order_pb2 import SortOrderEnum
from refb.merchant.v1.filters.bool_filter_pb2 import BoolFilter
from refb.merchant.v1.filters.market_type_filter_pb2 import MarketTypeFilter
from refb.merchant.v1.services.market_service_pb2 import ListMarketsRequest
from refb.merchant.v1.services.market_service_pb2_grpc import MarketServiceStub


API_HOST = 'api.refurbed.com:443'
API_TOKEN = 'f4e133d1-6cc4-4792-be5d-80a446a3d0dd'


class AuthMetadataPlugin(grpc.AuthMetadataPlugin):
    def __init__(self, token):
        super(AuthMetadataPlugin, self).__init__()
        self._token = token

    def __call__(self, context, callback):
        callback([('authorization', "Plain %s" % self._token)], None)


def create_channel(host, auth_token):
    refb_auth_plugin = AuthMetadataPlugin(auth_token)
    token_call_creds = grpc.metadata_call_credentials(refb_auth_plugin)

    ssl_creds = grpc.ssl_channel_credentials()
    creds = grpc.composite_channel_credentials(ssl_creds, token_call_creds)
    
    return grpc.secure_channel(host, creds)


def run():
    with create_channel(API_HOST, API_TOKEN) as channel:
        # creates the market service
        markets = MarketServiceStub(channel)
        
        # gets 3 site markets of type country, sorted by name
        req = ListMarketsRequest()
        req.pagination.limit = 3
        req.sort.by = ListMarketsRequest.Sort.By.NAME 
        req.sort.order = SortOrderEnum.SortOrder.DESC
        req.filter.is_site.value = True
        req.filter.type.any_of.extend([MarketTypeEnum.MarketType.COUNTRY])

        resp = markets.ListMarkets(req)
        for market in resp.markets:
            print(market)

        if resp.has_more:
            print("there are more results available")


if __name__ == '__main__':
    run()
```


## Contact

Please direct any questions to your account manager. If you don't have an
account manager assigned yet, you can contact
[integrations@refurbed.com](mailto:integrations@refurbed.com).
