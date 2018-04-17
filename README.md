Edit the following lines in the bottom of the file "api.php":

$api = new PHP_CRUD_API(array(
	'username'=>'xxx',
	'password'=>'xxx',
	'database'=>'xxx',
));
$api->executeCommand();
These are all the configuration options and their default values:

$api = new PHP_CRUD_API(array(
	'dbengine'=>'MySQL',
	'username'=>'root',
	'password'=>null,
	'database'=>false,
// for connectivity (defaults to localhost):
	'hostname'=>null,
	'port'=>null,
	'socket'=>null,
	'charset'=>'utf8',
// callbacks with their default behavior
	'table_authorizer'=>function($cmd,$db,$tab) { return true; },
	'record_filter'=>function($cmd,$db,$tab) { return false; },
	'column_authorizer'=>function($cmd,$db,$tab,$col) { return true; },
	'tenancy_function'=>function($cmd,$db,$tab,$col) { return null; },
	'input_sanitizer'=>function($cmd,$db,$tab,$col,$typ,$val) { return $val; },
	'input_validator'=>function($cmd,$db,$tab,$col,$typ,$val,$ctx) { return true; },
	'before'=>function(&$cmd,&$db,&$tab,&$id,&$in) { /* adjust array $in */ },
	'after'=>function($cmd,$db,$tab,$id,$in,$out) { /* do something */ },
// configurable options
	'allow_origin'=>'*',
	'auto_include'=>true,
// dependencies (added for unit testing):
	'db'=>null,
	'method'=>$_SERVER['REQUEST_METHOD'],
	'request'=>$_SERVER['PATH_INFO'],
	'get'=>$_GET,
	'post'=>file_get_contents('php://input'),
	'origin'=>$_SERVER['HTTP_ORIGIN'],
));
$api->executeCommand();
NB: The "socket" option is not supported by MS SQL Server. SQLite expects the filename in the "database" field.

Documentation
After configuring you can directly benefit from generated API documentation. On the URL below you find the generated API specification in Swagger 2.0 format.

http://localhost/api.php
Try the editor to quickly view it! Choose "File" > "Paste JSON..." from the menu.

Usage
You can do all CRUD (Create, Read, Update, Delete) operations and one extra List operation. Here is how:

List
List all records of a database table.

GET http://localhost/api.php/categories
Output:

{"categories":{"columns":["id","name"],"records":[[1,"Internet"],[3,"Web development"]]}}
List + Transform
List all records of a database table and transform them to objects.

GET http://localhost/api.php/categories?transform=1
Output:

{"categories":[{"id":1,"name":"Internet"},{"id":3,"name":"Web development"}]}
NB: This transform is CPU and memory intensive and can also be executed client-side (see: lib).

List + Filter
Search is implemented with the "filter" parameter. You need to specify the column name, a comma, the match type, another commma and the value you want to filter on. These are supported match types:

cs: contain string (string contains value)
sw: start with (string starts with value)
ew: end with (string end with value)
eq: equal (string or number matches exactly)
lt: lower than (number is lower than value)
le: lower or equal (number is lower than or equal to value)
ge: greater or equal (number is higher than or equal to value)
gt: greater than (number is higher than value)
bt: between (number is between two comma separated values)
in: in (number is in comma separated list of values)
is: is null (field contains "NULL" value)
You can negate all filters by prepending a 'n' character, so that 'eq' becomes 'neq'.

GET http://localhost/api.php/categories?filter=name,eq,Internet
GET http://localhost/api.php/categories?filter=name,sw,Inter
GET http://localhost/api.php/categories?filter=id,le,1
GET http://localhost/api.php/categories?filter=id,ngt,2
GET http://localhost/api.php/categories?filter=id,bt,1,1
GET http://localhost/api.php/categories?filter=categories.id,eq,1
Output:

{"categories":{"columns":["id","name"],"records":[[1,"Internet"]]}}
NB: You may specify table name before the field name, seperated with a dot.

List + Filter + Satisfy
Multiple filters can be applied by using "filter[]" instead of "filter" as a parameter name. Then the parameter "satisfy" is used to indicate whether "all" (default) or "any" filter should be satisfied to lead to a match:

GET http://localhost/api.php/categories?filter[]=id,eq,1&filter[]=id,eq,3&satisfy=any
GET http://localhost/api.php/categories?filter[]=id,ge,1&filter[]=id,le,3&satisfy=all
GET http://localhost/api.php/categories?filter[]=id,ge,1&filter[]=id,le,3&satisfy=categories.all
GET http://localhost/api.php/categories?filter[]=id,ge,1&filter[]=id,le,3
Output:

{"categories":{"columns":["id","name"],"records":[[1,"Internet"],[3,"Web development"]]}}
NB: You may specify "satisfy=categories.all,posts.any" if you want to mix "and" and "or" for different tables.

List + Column selection
By default all columns are selected. With the "columns" parameter you can select specific columns. Multiple columns should be comma separated. An asterisk ("*") may be used as a wildcard to indicate "all columns". Similar to "columns" you may use the "exclude" parameter to remove certain columns:

GET http://localhost/api.php/categories?columns=name
GET http://localhost/api.php/categories?columns=categories.name
GET http://localhost/api.php/categories?exclude=categories.id
Output:

{"categories":{"columns":["name"],"records":[["Web development"],["Internet"]]}}
NB: Columns that are used to include related entities are automatically added and cannot be left out of the output.

List + Order
With the "order" parameter you can sort. By default the sort is in ascending order, but by specifying "desc" this can be reversed:

GET http://localhost/api.php/categories?order=name,desc
GET http://localhost/api.php/posts?order[]=icon,desc&order[]=name
Output:

{"categories":{"columns":["id","name"],"records":[[3,"Web development"],[1,"Internet"]]}}
NB: You may sort on multiple fields by using "order[]" instead of "order" as a parameter name.

List + Order + Pagination
The "page" parameter holds the requested page. The default page size is 20, but can be adjusted (e.g. to 50):

GET http://localhost/api.php/categories?order=id&page=1
GET http://localhost/api.php/categories?order=id&page=1,50
Output:

{"categories":{"columns":["id","name"],"records":[[1,"Internet"],[3,"Web development"]],"results":2}}
NB: Pages that are not ordered cannot be paginated.

Create
You can easily add a record using the POST method (x-www-form-urlencoded, see rfc1738). The call returns the "last insert id".

POST http://localhost/api.php/categories
id=1&name=Internet
Output:

1
Note that the fields that are not specified in the request get the default value as specified in the database.

Create (with JSON object)
Alternatively you can send a JSON object in the body. The call returns the "last insert id".

POST http://localhost/api.php/categories
{"id":1,"name":"Internet"}
Output:

1
Note that the fields that are not specified in the request get the default value as specified in the database.

Create (with JSON array)
Alternatively you can send a JSON array containing multiple JSON objects in the body. The call returns an array of "last insert id" values.

POST http://localhost/api.php/categories
[{"name":"Internet"},{"name":"Programming"},{"name":"Web development"}]
Output:

[1,2,3]
This call uses a transaction and will either insert all or no records. If the transaction fails it will return 'null'.

Read
If you want to read a single object you can use:

GET http://localhost/api.php/categories/1
Output:

{"id":1,"name":"Internet"}
