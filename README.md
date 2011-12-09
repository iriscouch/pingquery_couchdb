# Ping the CouchDB query server

Pingquery is a plugin to help ascertain whether the query server is working normally. It provides a new URL where an admin can submit an expression in that language, (`2 + 2`), and compare it to the expected output (`4`).

## Good ping example

    POST /_pingquery/javascript HTTP/1.1
    Authorization: Basic <admin credentials>

    { "in" : "function() { return 'Hello world' }"
    , "out": "Hello world"
    }

    HTTP/1.1 200 OK
    Server: CouchDB/1.1.1 (Erlang OTP/R14B04)
    Date: Fri, 09 Dec 2011 01:39:02 GMT
    Content-Type: text/plain;charset=utf-8
    Content-Length: 35
    Cache-Control: must-revalidate

    {"ok":true,"match":"Hello, world"}

## Bad ping example

    POST /_pingquery/javascript HTTP/1.1
    Authorization: Basic <admin credentials>

    { "in" : "function() { return 'Foo' }"
    , "out": "Bar"
    }

    HTTP/1.1 500 Internal Server Error
    Server: CouchDB/1.1.1 (Erlang OTP/R14B04)
    Date: Fri, 09 Dec 2011 01:43:34 GMT
    Content-Type: text/plain;charset=utf-8
    Content-Length: 75
    Cache-Control: must-revalidate

    {"error":"bad_ping","reason":"no_match","expected":"Bar","received":"Foo"}

## Usage

I like to use cURL from a cron job to detect problems:

    $ curl --silent --fail -XPOST -HContent-Type:application/json -o /dev/null \
           https://admin:secret@example.iriscouch.com/_pingquery/javascript    \
           -d '{"in":"function() { return typeof log }", "out":"function"}'
    $ echo $?
    0

## Compatibility

Pingquery works with all query server languages: JavaScript, CoffeeScript, Native (Erlang), Python, Ruby, etc.

To execute a ping, the plugin tricks the query server into running a standard `_show` function, then examines the output.

## Building

If you hate Freedom, just sign up for an [Iris Couch][ic] account, since they run hosted CouchDB with this plugin.

To build for yourself, use Build CouchDB: https://github.com/iriscouch/build-couchdb

    rake plugin="git://github.com/iriscouch/pingquery_couchdb origin/master"

[ic]: http://www.iriscouch.com/service

## Installation

Build CouchDB already took care of this step!

## Development

This is what I do. It's not perfect but barely harder than building a fork of CouchDB, and it allows 1.0.x, 1.1.x, as well as trunk support.

This example assumes three Git checkouts, side-by-side:

* `couchdb` - Perhaps a trunk checkout, but could be any tag or branch
* `build-couchdb` - For the boring Couch dependencies
* `pingquery_couchdb` - This code

### Build dependencies plus couch

Feel free to skip this if you are Randall Leeds. (Hi, Randall!)

    cd couchdb
    rake -f ../build-couchdb/Rakefile couchdb:configure
    # Go get coffee. This builds the deps, then runs the boostrap and configure scripts
    make dev

Next, teach rebar where to find CouchDB, and teach Erlang and Couch where to find this plugin.

    cd ../pingquery_couchdb
    export ERL_COMPILER_OPTIONS='[{i, "../couchdb/src/couchdb"}]'
    export ERL_ZFLAGS="-pz $PWD/ebin"
    ln -s "../../../../pingquery_couchdb/etc/couchdb/default.d/pingquery.ini" ../couchdb/etc/couchdb/default.d

You're ready. Run this every time you change the code:

    ./rebar compile && ../couchdb/utils/run -i
