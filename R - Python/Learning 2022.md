## Thursday, June 16

Learned: Function `*args` and `**kwargs`

In Python, functions can be defined with multiple unknown arguments (different “arity” also called “variadic parameters).

For example, you could have an `add` function that could take in any number of arguments that then sums all those arguments and returns the sum total.

The following example shows how you’d define known parameters:

```python
def add(operand1: int, operand2: int)
  return operand1 + operand2
```

With `*args`, we can modify the function to take any number of non-keyworded arguments:

```python
def add(*args)
	total = 0
  for (arg in args)
    total += arg
  return total
```

`**kwargs` offer an additional way to provide multiple unknown arguments that are keyworded. “Keyworded” here means that the argument has a name like so:

```python
def some_func(**kwargs):
	for key, value in kwargs.items():
    print(f"key: {key}, value: {value}")

some_func(arg1 = "Hello", arg2 = "World")

# Prints
# "key: arg1, value: Hello"
# "key: arg2, value: World"
```

## Thursday, July 14

Running multiple coroutines using the asyncio event loop. Elastic Search example:

```python
def __get_users(self):
    users = User.objects.order_by('id').iterator()
    for user in users:
        user_doc = UserDocument()
        user_doc = _set_doc_obj_attributes(user_doc, user)
        action = user_doc.to_dict(include_meta=True)
        yield action

def __get_events(self):
    events = Event.objects.order_by('id').iterator()
    for event in events:
        event_doc = EventDocument()
        event_doc = _set_doc_obj_attributes(event_doc, event)
        action = event_doc.to_dict(include_meta=True)
        yield action

def __get_orders(self):
    orders = Order.objects.order_by('id').iterator()
    for order in orders:
        order_doc = OrderDocument()
        order_doc = _set_doc_obj_attributes(order_doc, order)
        action = order_doc.to_dict(include_meta=True)
        yield action

async def index_users(self):
    await async_bulk(self._es, self.__get_users(), chunk_size=5000)

async def index_events(self):
    await async_bulk(self._es, self.__get_events(), chunk_size=100)

async def index_orders(self):
    await async_bulk(self._es, self.__get_orders(), chunk_size=100)

# Important part here
input_coroutines = [self.index_users(), self.index_orders(), self.index_events()]
res = gather(*input_coroutines, return_exceptions=True)
loop = get_event_loop()
loop.run_until_complete(res)
```

## Tuesday, July 19
```python
from django.core.management.base import BaseCommand

from asyncio import gather, get_event_loop
from botocore.auth import SigV4Auth
from botocore.awsrequest import AWSRequest
from collections import deque
from dataclasses import dataclass
from elasticsearch import AsyncElasticsearch, Elasticsearch, AIOHttpConnection, RequestsHttpConnection
from elasticsearch.helpers import async_bulk, bulk, parallel_bulk
from elasticsearch_dsl import Index
from requests_aws4auth import AWS4Auth
from urllib.parse import urlencode

from app.models.event import Event, Order
from app.models.user import User
from search.elasticsearch_client import _set_doc_obj_attributes
from search.elasticsearch_documents import UserDocument, EventDocument, OrderDocument
from settings import AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, ELASTICSEARCH_HOST

# Re-populate indices

# Attempt #1: Doesn't work, hangs the shell
# for user in all_users:
#   create_or_update_user(user)

# for event in all_events:
#   create_or_update_event(event)

# for order in all_orders:
#   create_or_update_order(order)

# Attempt #2: Batch load fetching a queryset 2000 records at a time
# def batch_qs(qs, batch_size=1000):
#     """
#     Returns a (start, end, total, queryset) tuple for each batch in the given
#     queryset.

#     Usage:
#         # Make sure to order your querset
#         article_qs = Article.objects.order_by('id')
#         for start, end, total, qs in batch_qs(article_qs):
#             print "Now processing %s - %s of %s" % (start + 1, end, total)
#             for article in qs:
#                 print article.body
#     """
#     total = qs.count()
#     for start in range(0, total, batch_size):
#         end = min(start + batch_size, total)
#         yield (start, end, total, qs[start:end])

# for start, end, total, users_chunk in batch_qs(all_users, 2000):
#     print(f"Now processing {start+1} - {end} of {total}")
#     for user in users_chunk:

# Attempt #3: Use bulk helper APIs provided by elasticsearch library

@dataclass
class Credential:
    access_key = ""
    secret_key = ""
    token = None

class AWSAuthAIOHttpConnection(AIOHttpConnection):
    """Enable AWS Auth with AIOHttpConnection for AsyncElasticsearch

    The AIOHttpConnection class built into elasticsearch-py is not currently
    compatible with passing AWSAuth as the `http_auth` parameter, as suggested
    in the docs when using AWSAuth for the non-async RequestsHttpConnection class:
    https://docs.aws.amazon.com/opensearch-service/latest/developerguide/request-signing.html#request-signing-python

    To work around this we patch `AIOHttpConnection.perform_request` method to add in
    AWS Auth headers before making each request.

    This approach was synthesized from
    * https://stackoverflow.com/questions/38144273/making-a-signed-http-request-to-aws-elasticsearch-in-python
    * https://github.com/DavidMuller/aws-requests-auth
    * https://github.com/jmenga/requests-aws-sign
    * https://github.com/byrro/aws-lambda-signed-aiohttp-requests
    """

    def __init__(self, *args, aws_region=None, **kwargs):
        super().__init__(*args, **kwargs)
        self.aws_region = aws_region
        credential = Credential()
        credential.access_key = AWS_ACCESS_KEY_ID
        credential.secret_key = AWS_SECRET_ACCESS_KEY
        self.auther = SigV4Auth(credential, "es", self.aws_region)

    def _make_full_url(self, url: str, params=None) -> str:
        # These steps are copied from the parent class' `perform_request` implementation.
        # The ElasticSearch client only passes in the request path as the url,
        # and that partial url format is rejected by the `SigV4Auth` implementation
        query_string = urlencode(params) if params else None
        full_url = self.url_prefix + self.host + url
        if query_string:
            full_url = "%s?%s" % (full_url, query_string)
        return full_url

    async def perform_request(
        self, method, url, params=None, body=None, timeout=None, ignore=(), headers=None
    ):
        full_url = self._make_full_url(url)
        if headers is None:
            headers = {}

        # this request object won't be used, we just want to copy its auth headers
        # after `SigV4Auth` processes it and adds the headers
        _request = AWSRequest(
            method=method, url=full_url, headers=headers, params=params, data=body
        )
        self.auther.add_auth(_request)
        headers.update(_request.headers.items())
        # passing in the original `url` param here works too
        return await super().perform_request(
            method, url, params, body, timeout, ignore, headers
        )

# awsauth = AWS4Auth(AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, 'us-west-2', 'es')
# es = AsyncElasticsearch(
#             [ELASTICSEARCH_HOST],
#             use_ssl=True,
#             verify_certs=True,
#             connection_class=AWSAuthAIOHttpConnection,
#             aws_region="us-west-2"
#         )

# def sync_gendata(user):
#     user_doc = UserDocument()
#     user_doc = _set_doc_obj_attributes(user_doc, user)
#     return user_doc.to_dict(include_meta=True)

# bulk(es, (sync_gendata(user) for user in User.objects.iterator()), chunk_size=2000)

# def gendata():
#     users = User.objects.order_by('id')[:100000].iterator()
#     for user in users:
#         user_doc = UserDocument()
#         user_doc = _set_doc_obj_attributes(user_doc, user)
#         action = user_doc.to_dict(include_meta=True)
#         yield action

# async def run():
#     await async_bulk(es, gendata(), chunk_size=5000)

# asyncio.get_event_loop().run_until_complete(run())

class Command(BaseCommand):
    help = "Recreates the Elastic Search indices for User, Event and Order models."

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._es = AsyncElasticsearch(
            [ELASTICSEARCH_HOST],
            use_ssl=True,
            verify_certs=True,
            connection_class=AWSAuthAIOHttpConnection,
            aws_region="us-west-2"
        )
        self._ses = Elasticsearch(
            hosts=[ELASTICSEARCH_HOST],
            http_auth=AWS4Auth(AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, 'us-west-2', 'es'),
            timeout=10, use_ssl=True, verify_certs=True,
            connection_class=RequestsHttpConnection
        )

    def __get_users(self):
        users = User.objects.order_by('id').iterator()
        for user in users:
            user_doc = UserDocument()
            user_doc = _set_doc_obj_attributes(user_doc, user)
            action = user_doc.to_dict(include_meta=True)
            yield action

    def __get_events(self):
        events = Event.objects.order_by('id')[:100000].iterator()
        for event in events:
            event_doc = EventDocument()
            event_doc = _set_doc_obj_attributes(event_doc, event)
            action = event_doc.to_dict(include_meta=True)
            yield action

    def __get_orders(self):
        orders = Order.objects.order_by('id').iterator()
        for order in orders:
            order_doc = OrderDocument()
            order_doc = _set_doc_obj_attributes(order_doc, order)
            action = order_doc.to_dict(include_meta=True)
            yield action

    async def index_users(self):
        await async_bulk(self._es, self.__get_users(), chunk_size=5000)

    async def index_events(self):
        await async_bulk(self._es, self.__get_events(), chunk_size=5000)

    async def index_orders(self):
        await async_bulk(self._es, self.__get_orders(), chunk_size=5000)

    async def index_models(self):
        await gather(self.index_users(), self.index_events(), self.index_orders())

    def handle(self, **options):
        # Get indices
        # user_index = Index('user')
        event_index = Index('event')
        # order_index = Index('order')

        # Destroy indices
        # order_index.delete()
        event_index.delete()
        # user_index.delete()

        # Create indices
        # user_index.document(UserDocument)
        event_index.document(EventDocument)
        # order_index.document(OrderDocument)
        # user_index.create()
        event_index.create()
        # order_index.create()

        # Running multiple concurrent tasks
        # loop = get_event_loop()
        # try:
        #     loop.run_until_complete(self.index_models())
        # finally:
        #     loop.close()

        # Running one task
        loop = get_event_loop()
        try:
            loop.run_until_complete(self.index_events())
        finally:
            loop.close()

        # Synchronous Bulk
        # bulk(self._ses, self.__get_orders(), chunk_size=5000)
```