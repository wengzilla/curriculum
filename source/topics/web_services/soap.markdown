---
layout: page
title: Consuming SOAP with Savon
section: Web Services
---

Sometimes you've got to use SOAP. Let's take a look at the Savon gem and how it allows us to interact with SOAP services.

## Savon

Savon can be installed by adding `gem savon` to your `Gemfile` and running `bundle`.

### Configuring the Client

The easiest way to get started is to use a WSDL document.

```ruby
client = Savon::Client.new do |wsdl|
  wsdl.document = "http://service.example.com?wsdl"
end
```

`Savon::Client.new` accepts a block where you configure the client. In order to use the wsdl and an http proxy, you can specify two arguments:

```ruby
client = Savon::Client.new do |wsdl, http|
  wsdl.document = "http://service.example.com?wsdl"
  http.proxy = "http://proxy.example.com"
end
```

#### With a Local WSDL

To use a local WSDL, you specify the path to the file instead of the remote location.

```ruby
client = Savon::Client.new do
  wsdl.document = File.expand_path("../wsdl/ebay.xml", __FILE__)
end
```

#### Without a WSDL

To use Savon without a WSDL, you initialize a client by setting the SOAP endpoint and target namespace.

```ruby
client = Savon::Client.new do |wsdl|
  wsdl.endpoint = "http://service.example.com"
  wsdl.namespace = "http://v1.example.com"
end
```

#### Checking the Results

With the client set up, you can now see what Savon knows about your service:

```ruby
# the target namespace
client.wsdl.namespace     # => "http://v1.example.com"

# the SOAP endpoint
client.wsdl.endpoint      # => "http://service.example.com"

# available SOAP actions
client.wsdl.soap_actions  # => [:create_user, :get_user, :get_all_users]

# the raw document
client.wsdl.to_xml        # => "<wsdl:definitions...>"
```

The remote service probably uses `CamelCase` names for actions and params, but Savon maps those to `snake_case` Symbols for you.

### Preparing for HTTP

Savon uses [HTTPI](http://rubygems.org/gems/httpi) to execute GET requests for WSDL documents and POST requests for SOAP requests. HTTPI is an interface to HTTP libraries like Curl and Net::HTTP.

The library comes with a [`HTTPI::Request`](http://github.com/rubiii/httpi/blob/master/lib/httpi/request.rb) object which you can access through the client. 

If your service relies on cookies to handle sessions, you can grab the cookie from the [`HTTPI::Response`](http://github.com/rubiii/httpi/blob/master/lib/httpi/response.rb) and set it for subsequent requests.

```ruby
client.http.headers["Cookie"] = response.http.headers["Set-Cookie"]
```

### WSSE authentication

Savon comes with [`Savon::WSSE`](http://github.com/rubiii/savon/blob/master/lib/savon/wsse.rb) for you to use `wsse:UsernameToken` authentication.

```ruby
client.wsse.credentials "username", "password"
```

Or `wsse:UsernameToken` digest authentication:

```ruby
client.wsse.credentials "username", "password", :digest
```

### Executing SOAP requests

To execute SOAP requests, you use the `Savon::Client#request` method. Here's a very basic example of executing a SOAP request to a `get_all_users` action.

```ruby
response = client.request :get_all_users
```

Here `:get_all_users` will be converted by Savon to `"getAllUsers"` on the remote service. If you do not want Savon to manipulate the case/style of our action, use a string:

```ruby
response = client.request "GetAllUsers"
```

### Working with SOAP

To interact with your service, you probably need to specify some SOAP-specific options. The `#request` method is the second important method to accept a block and lets you access the
following objects.

    [soap, wsdl, http, wsse]

The `soap` variable, an instance of [`Savon::SOAP::XML`](http://github.com/rubiii/savon/blob/master/lib/savon/soap/xml.rb), can only be accessed inside the block and Savon creates a new soap object for every request.

#### SOAP Versioning

Savon by default expects your services to be based on SOAP 1.1. For SOAP 1.2 services, you can set the SOAP version per request.

```ruby
response = client.request :get_user do
  soap.version = 2
end
```

#### Namespacing

If you don't pass a namespace to the `#request` method, Savon will attach the target namespaces to `"xmlns:wsdl"`. If you pass a namespace, Savon will use it instead of the default.

```ruby
client.request :v1, :get_user
```

Which will result in:

```xml
<env:Envelope
    xmlns:env="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:xsd="http://www.w3.org/2001/XMLSchema"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:v1="http://v1.example.com">
  <env:Body>
    <v1:GetUser>
  </env:Body>
</env:Envelope>
```

You can set and overwrite namespaces as a Hash:

```ruby
# setting a namespace
soap.namespaces["xmlns:g2"] = "http://g2.example.com"

# overwriting "xmlns:wsdl"
soap.namespaces["xmlns:wsdl"] = "http://ns.example.com"
```

### A Little Interaction

To call the `get_user` action of a service and pass the ID of the user to return, you can use a Hash for the SOAP body.

```ruby
response = client.request :get_user do
  soap.body = { :id => 1 }
end
```

The Hash is translated to XML using [Gyoku](http://rubygems.org/gems/gyoku).

```ruby
soap.body = {
  :first_name => "The",
  :last_name  => "Hoff",
  "FAME"      => ["Knight Rider", "Baywatch"]
}
```

Again, Symbol keys will be converted to `lowerCamelCase` and String keys won't be touched. The previous example generates the following XML.

```xml
<env:Envelope
    xmlns:env="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:wsdl="http://v1.example.com">
  <env:Body>
    <wsdl:CreateUser>
      <firstName>The</firstName>
      <lastName>Hoff</lastName>
      <FAME>Knight Rider</FAME>
      <FAME>Baywatch</FAME>
    </wsdl:CreateUser>
  </env:Body>
</env:Envelope>
```

#### Ordering Elements

Some services require the XML elements to be in a specific order. If you are on Ruby 1.8, you can not be sure about the order of Hash elements. 

This will cause problems unless you specify the correct order using an Array under a special `:order!` key.

```ruby
{ :last_name => "Hoff", :first_name => "The", :order! => [:first_name, :last_name] }
```

This will make sure, that the `lastName` tag follows the `firstName`.

#### Arguments to Tags

Assigning arguments to XML tags using a Hash requires another Hash under
an `:attributes!` key containing a key matching the XML tag and the Hash of attributes to add.

```ruby
{ :city => nil, :attributes! => { :city => { "xsi:nil" => true } } }
```

This example will be translated to the following XML.

```xml
<city xsi:nil="true"></city>
```

#### Arguments with XML Builder

An alternative approach is to pass a block to the `Savon::SOAP::XML#xml`
method. Savon will then yield a `Builder::XmlMarkup` instance for you to use.

```ruby
soap.xml do |xml|
  xml.firstName("The")
  xml.lastName("Hoff")
end
```

#### Arguments with Plain Strings

You can also create and use a plain String:

```ruby
soap.body = "<firstName>The</firstName><lastName>Hoff</lastName>"
```

#### Headers

Besides the body element, SOAP requests can also contain a header with additional information. Savon sees this header as just another Hash following the same conventions as the SOAP body Hash.

```ruby
soap.header = { "SecretKey" => "secret" }
```

### Handling the Response

`Savon::Client#request` returns a
[`Savon::SOAP::Response`](http://github.com/rubiii/savon/blob/master/lib/savon/soap/response.rb) which convert to a Hash:

```ruby
response.to_hash  # => { :response => { :success => true, :name => "John" } }
```

Or you can interact with it as XML:

```ruby
response.to_xml  # => "<response><success>true</success><name>John</name></response>"
```

The response also contains the [`HTTPI::Response`](http://github.com/rubiii/httpi/blob/master/lib/httpi/response.rb)
which contains information about the HTTP response.

```ruby
response.http  # => #<HTTPI::Response:0x1017b4268 ...
```

### Ecosystem 

If you're building SOAP/Savon powered systems, here are a few coordinating libraries that might be helpful:

* [Savon::Model](http://rubygems.org/gems/savon_model) creates SOAP service oriented models
* [Savon::Spec](http://rubygems.org/gems/savon_spec) helps you test your SOAP requests

## References

* Savon Gem Source: http://github.com/rubiii/savon
* Savon API Documentation: http://rubydoc.info/gems/savon/frames
* HTTPI Gem: http://rubygems.org/gems/httpi
* Gyoku Gem: http://rubygems.org/gems/gyoku