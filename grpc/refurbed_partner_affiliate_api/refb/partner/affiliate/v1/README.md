# Refurbed Affiliate Partner API

The refurbed affiliate partner API is based on [gRPC](https://grpc.io). There is
also  an HTTP version of this API available.


## Key concepts

Please refer to the fully documented proto files for more details about which
methods and objects are supported in detail. Start with the services to see what
operations are available to you. You can find I/O objects in the services files.
The main data objects are defined in the `resources/` directory.


### HTTP API

If you prefer to use the HTTP version of this API, please refer to the 
OpenAPI definition
(`./refurbed_partner_affiliate_api/refb/partner/affiliate/v1/api.swagger.json`).
You can use it to automatically generate a client. To display end-user 
documentation from the JSON, you can run the following command and visit 
http://localhost:9999

```
$ docker run -p 9999:8080 \
    -v `(pwd)`/refurbed_partner_affiliate_api/refb/partner/affiliate/v1:/openapi \
    -e SWAGGER_JSON=/openapi/api.swagger.json \
    swaggerapi/swagger-ui
```

### Data types

Monetary amounts as well as other decimal types which require exact precision
(e.g. exchange rates) are encoded as strings. They use a decimal point as
separator (e.g. `123.45`).

Monetary amounts support a precision of up to 2 decimal digits
(valid: `123`, `123.45`; invalid: `123.456`).

### Markets

Market represents an area in which the instances are being sold. Instances 
offered are always linked to a specific market, which needs to be a site market.
When retrieving product or instances you must specify which site market you are
interested. You can use the `MarketService` to retrieve supported markets, use
`is_site` filter retrieve only site markets.

### Products

A product represents a generic item of which specific variants, called instances,
are sold on refurbed marketplace by different merchants. Use the `ProductService`
to list products sold on refurbed marketplace. `ListProducts()` offers a variety
of filters to allow you to specify exactly what you want to receive.

Each product has BuyBox data attached which shows the best offer currently
available for all of the product's instances.

### Instances

Product variants at refurbed are called instances. You can use the 
`InstanceService` to retrieve instances matching filters or a single instance
identifier either by GTIN or refurbed instance id.

Each instance has BuyBox data attached which shows the best offer currently
available for this instance.

### Buybox

Buybox data, defines the price and conditions (grading, warranty) by which
an instance is being sold on the marketplace. It also includes a link to be used
redirect to for the customer to complete the purchase.


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
    -v `pwd`/refurbed_partner_affiliate_api:/defs \
    namely/protoc-all \
    -i . \
    -d refb/ \
    -l python
```

This will create a new directory `./refurbed_partner_affiliate_api/gen/pb_python`
with the relevant client code inside. You can copy the `refb` directory into 
your project.


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
from refb.partner.affiliate.v1.enums.market_type_pb2 import MarketTypeEnum
from refb.partner.affiliate.v1.enums.sort_order_pb2 import SortOrderEnum
from refb.partner.affiliate.v1.filters.bool_filter_pb2 import BoolFilter
from refb.partner.affiliate.v1.filters.market_type_filter_pb2 import MarketTypeFilter
from refb.partner.affiliate.v1.services.market_service_pb2 import ListMarketsRequest
from refb.partner.affiliate.v1.services.market_service_pb2_grpc import MarketServiceStub


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

Please direct any questions to your partnership manager. If you don't have a
partnership manager assigned yet, you can contact
[integrations@refurbed.com](mailto:integrations@refurbed.com).
