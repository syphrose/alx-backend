## PAGINATION
It is the process of dividing a document into discrete pages, either electronic pages or printed pages.
 
REST API pagination is the process of splitting data sets into discrete pages – a set of paginated endpoints. An API call to a paginated endpoint is a paginated request. There are three most common pagination techniques that define the way to make paginated requests: Cursor pagination.

 # Hypermedia as the engine of application state(HATEOAS)

HATEOAS is a constraint of the REST application architecture that distinguishes it from other network application architectures.

With HATEOAS, a client interacts with a network application whose application servers provide information dynamically through hypermedia. A REST client needs little to no prior knowledge about how to interact with an application or server beyond a generic understanding of hypermedia.

By contrast, clients and servers in Common Object Request Broker Architecture (CORBA) interact through a fixed interface shared through documentation or an interface description language (IDL).

The restrictions imposed by HATEOAS decouple client and server. This enables server functionality to evolve independently.

# FILTERING
URL parameters is the easiest way to add basic filtering to REST APIs. If you have an /items endpoint which are items for sale, you can filter via the property name such as GET /items?state=active or GET /items?state=active&seller_id=1234. However, this only works for exact matches. What if you want to do a range such as a price or date range?

The problem is URL parameters only have a key and a value but filters are composed of three components:

The property or field name
The operator such as eq, lte, gte
The filter value
There are various ways to encode three components into URL param key/values.

# LHS Brackets
One way to encode operators is the use of square brackets [] on the key name. For example, GET /items?price[gte]=10&price[lte]=100 would find all the items where the price is greater than or equal to 10, but less than or equal to 100.

We can have as many operators as needed such as [lte], [gte], [exists], [regex], [before], and [after].

LHS Brackets are a little harder to parse on server side, but provides greater flexibility in what the filter value is for clients. No need to handle special characters differently.

# Benefits
Ease of use for clients. There are many query string parsing libraries available that easily encode nested JSON objects into square brackets. qs is one such library that automatically encodes/decodes square brackets:

var qs = require('qs');
var assert = require('assert');

assert.deepEqual(qs.parse('price[gte]=10&price[lte]=100'), {
    price: {
        gte: 10,
        lte: 100
    }
});
Simple to parse on server side. The URL parameter key contains both the field name and operator. Easy to GROUP BY (property name, operator) without looking at the URL parameter values.

No need to escape special characters in the filter value when operator is taken as a literal filter term. This is especially true when your filters include additional custom metadata fields that your user may set.

# Downsides
May require more work on server side to parse and group the filters. You may have to write a custom URL parameter binder or parser to split the query string key into two components: The field name and the operator. You would then need to GROUP BY (property name, operator).

Special characters in variable names can be awkward. You may have to write a custom binder to split the query string key into two components: The field name and the operator.

Hard to manage custom combinational filters. Multiple filters with the same property name and operator result in an implicit AND. What if the API user wanted to OR the filters instead. i.e. find all items where price is less than 10 OR greater than 100?

# RHS Colon
Similar to the bracket approach, you can design an API to take the operator on the RHS instead of LHS. For example, GET /items?price=gte:10&price=lte:100 would find all the items where the price is greater than or equal to 10, but less than or equal to 100.

# Benefits
Easiest to parse on server side especially if duplicate filters are not supported. No custom binders are needed. Many API frameworks already handle URL parameter arrays. Multiple price filters will be under the same variable ‘price’ which may be a Sequence or Map.

# Downsides
Literal values need special handling. For example, GET /items?user_id=gt:100 would translate to find all items where the user_id is greater than 100. However, what if we want to find all items where the user_id equals gt:100 as that could be a valid id?

# Search Query Param
If you require search on your endpoint, you can add support for filters and ranges directly with the search parameter. If you’re already using ElasticSearch or other Lucene based technology, you could support the Lucene syntax or ElasticSearch Simple Query Strings directly.

For example, we could search items for those that contain the terms red chair and the price is greater than or equal to 10 and less than or equal to 100: GET /items?q=title:red chair AND price:[10 TO 100]

Such APIs can allow fuzziness matching, boosting certain terms, and more.

# Benefits
Most flexible queries for API users

Almost no parsing required on backend, can pass directly to search engine or database (Just be careful of sanitizing inputs for security)

# Downsides
Harder for beginners to start working with the API. Need to become familiar Lucene syntax.

Full-text search doesn’t make sense for all resources. For example, Fuzziness and term boosting doesn’t make sense for time series metric data.

Requires URL percent encoding which makes using cURL or Postman more complicated.

What is Moesif? Moesif is the most advanced REST API analytics platform used by Thousands of platformsto understand how your customers use your APIs and which filters they use most. Moesif has SDKs and plugins for popular API gateways such as Kong, Tyk and more.

# Pagination
Most endpoints that returns a list of entities will need to have some sort of pagination.

Without pagination, a simple search could return millions or even billions of hits causing extraneous network traffic.

Paging requires an implied ordering. By default this may be the item’s unique identifier, but can be other ordered fields such as a created date.

# Offset Pagination
This is the simplest form of paging. Limit/Offset became popular with apps using SQL databases which already have LIMIT and OFFSET as part of the SQL SELECT Syntax. Very little business logic is required to implement Limit/Offset paging.

Limit/Offset Paging would look like GET /items?limit=20&offset=100. This query would return the 20 rows starting with the 100th row.

# Example
(Assume the query is ordered by created date descending)

Client makes request for most recent items: GET /items?limit=20
On scroll/next page, client makes second request GET /items?limit=20&offset=20
On scroll/next page, client makes third request GET /items?limit=20&offset=40
As a SQL statement, the third request would look like:

SELECT
    *
FROM
    Items
ORDER BY Id
LIMIT 20
OFFSET 40;
Benefits
Easiest to implement, almost no coding required other than passing parameters directly to SQL query.

Stateless on the server.

Works regardless of custom sort_by parameters.

# Downsides
Not performant for large offset values. Let’s say you perform a query with a large offset value of 1000000. The database needs to scan and count rows starting with 0, and will skip (i.e. throw away data) for the first 1000000 rows.

Not consistent when new items are inserted to the table (i.e. Page drift) This is especially noticeable when we are ordering items by newest first. Consider the following that orders by decreasing Id:

Query GET /items?offset=0&limit=15
10 new items added to the table
Query GET /items?offset=15&limit=15 The second query will only return 5 new items, as adding 10 new items moved the offset back by 10 items. To fix this, the client would really need to offset by 25 for the second query GET /items?offset=25&limit=15, but the client couldn’t possibly know other objects being inserted into the table.
Even with limitations, offset paging is easy to implement and understand and can be used in applications where the data set has a small upper bounds.

# Keyset Pagination
Keyset pagination uses the filter values of the last page to fetch the next set of items. Those columns would be indexed.

Example
(Assume the query is ordered by created date descending)

Client makes request for most recent items: GET /items?limit=20
On scroll/next page, client finds the minimum created date of 2021-01-20T00:00:00 from previously returned results. and then makes second query using date as a filter: GET /items?limit=20&created:lte:2021-01-20T00:00:00
On scroll/next page, client finds the minimum created date of 2021-01-19T00:00:00 from previously returned results. and then makes third query using date as a filter: GET /items?limit=20&created:lte:2021-01-19T00:00:00
SELECT
    *
FROM
    Items
WHERE
  created <= '2021-01-20T00:00:00'
ORDER BY Id
LIMIT 20
Benefits
Works with existing filters without additional backend logic. Only need an additional limit URL parameter.

Consistent ordering even when newer items are inserted into the table. Works well when sorting by most recent first.

Consistent performance even with large offsets.

# Downsides
Tight coupling of paging mechanism to filters and sorting. Forces API users to add filters even if no filters are intended.

Does not work for low cardinality fields such as enum strings.

Complicated for API users when using custom sort_by fields as the client needs to adjust the filter based on the field used for sorting.

Keyset pagination can work very well for data with a single natural high cardinality key such as time series or log data which can use a timestamp.

# Seek Pagination
Seek Paging is an extension of Keyset paging. By adding an after_id or start_id URL parameter, we can remove the tight coupling of paging to filters and sorting. Since unique identifiers are naturally high cardinality, we won’t run into issues unlike if sorting by a low cardinality field like state enums or category name.

The problem with seek based pagination is it’s hard to implement when a custom sort order is needed.

Example
(Assume the query is ordered by created date ascending)

Client makes request for most recent items: GET /items?limit=20
On scroll/next page, client finds the last id of ‘20’ from previously returned results. and then makes second query using it as the starting id: GET /items?limit=20&after_id=20
On scroll/next page, client finds the last id of ‘40’ from previously returned results. and then makes third query using it as the starting id: GET /items?limit=20&after_id=40
Seek pagination can be distilled into a where clause

SELECT
    *
FROM
    Items
WHERE
  Id > 20
LIMIT 20
The above example works fine if ordering is done by id, but what if we want to sort by an email field? For each request, the backend needs to first obtain the email value for the item who’s identifier matches the after_id. Then, a second query is performed using that value as a where filter.

Let’s consider the query GET /items?limit=20&after_id=20&sort_by=email, the backend would need two queries. The first query could be O(1) lookup with hash tables though to get the email pivot value. This is fed into the second query to only retrieve items whose email is after our after_email. We sort by both columns, email and id to ensure we have a stable sort incase two emails are the same. This is critical for lower cardinality fields.

1.

SELECT
    email AS AFTER_EMAIL
FROM
    Items
WHERE
  Id = 20
2.

SELECT
    *
FROM
    Items
WHERE
  Email >= [AFTER_EMAIL]
ORDER BY Email, Id
LIMIT 20
Benefits
No coupling of pagination logic to filter logic.

Consistent ordering even when newer items are inserted into the table. Works well when sorting by most recent first.

Consistent performance even with large offsets.

# Downsides
More complex for backend to implement relative to offset based or keyset based pagination

If items are deleted from the database, the start_id may not be a valid id.

Seek paging is a good overall paging strategy and what we implemented on the Moesif Public API. It requires a little more work on the backend, but ensures there isn’t additional complexity added to clients/users of the API while staying performant even with larger seeks.

# Sorting
Like filtering, sorting is an important feature for any API endpoint that returns a lot of data. If you’re returning a list of users, your API users may want to sort by last modified date or by email.

To enable sorting, many APIs add a sort or sort_by URL parameter that can take a field name as the value.

However, good API designs give the flexibility to specify ascending or descending order. Like filters, specifying the order requires encoding three components into a key/value pair.

Example formats
GET /users?sort_by=asc(email) and GET /users?sort_by=desc(email)

GET /users?sort_by=+email and GET /users?sort_by=-email

GET /users?sort_by=email.asc and GET /users?sort_by=email.desc

GET /users?sort_by=email&order_by=asc and GET /users?sort_by=email&order_by=desc

Multi-Column Sort
It’s not recommended to use the last design where sort and order are not paired. You may eventually allow sorting by two or more columns:

SELECT
    email
FROM
    Items
ORDER BY Last_Modified DESC, Email ASC
LIMIT 20
To encode this multi-column sort, you could allow multiple field names such as

GET /users?sort_by=desc(last_modified),asc(email) or

GET /users?sort_by=-last_modified,+email

If the sort field and ordering were not paired, URL parameter ordering needs to be preserved; otherwise, it’s ambiguous what ordering should be paired with what field name. However, many server side frameworks may not preserve ordering once deserialized into a map.

You also have to ensure URL parameter ordering is considered for any cache keys, but this will put pressure on cache sizes.

# Conclusion
Good API design is a critical component for your Developer Experience (DX). API specifications can outlast many underlying server implementations which requires thinking about future use cases for your API.

What Is Tiered Pricing? The Ultimate Guide
Find out how tiered pricing can be a versatile and effective strategy for catering to diverse customer needs while maximizing revenue.
