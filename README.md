# Blink

Remote Procedure Call Server and Client for Crystal. Implements msgpack-rpc protocall. Designed to be reliable and stable (catch every possible protocall/socket errors). It also quite performant: benchmark shows ~ 200K rps in pool mode (single server core, single client core).

## Installation

Add this to your application's `shard.yml`:

```yaml
dependencies:
  blink:
    github: grkek/blink
```

## Usage

```crystal
require "blink"

# Example run server and client.

class Handler
  # When including Blink::Protocol, all public instance methods inside class,
  # would be exposed to external rpc call.
  # Each method should define type for each argument, and also return type.
  # (Types of arguments should supports MessagePack::Serializable).
  # Instance of this class created on server for each call.
  include Blink::Protocol

  def bla(x : Int32, y : String) : Float64
    x * y.to_f
  end
end

spawn do
  # running RPC server on 9000 port in background fiber
  Handler::Server.new("127.0.0.1", 9000).run
end

# wait until server up
sleep 0.1

# create rpc client
client = Handler::Client.new("127.0.0.1", 9000)
result = client.bla!(3, "5.5") # here can raise Blink::Errors
p result # => 16.5
```

#### When client code have no access to server Protocol, you can call raw requests:
```crystal
require "blink"

client = Blink::Client.new("127.0.0.1", 9000)
result = client.request!(Float64, :bla, 3, "5.5") # here can raise Blink::Errors
p result # => 16.5
```

#### When you dont want to raises on problems, you can check result by yourself:
```crystal
require "blink"

client = Blink::Client.new("127.0.0.1", 9000)
result = client.request(Float64, :bla, 3, "5.5") # no raise on error
if result.ok?
  p result.value! # => 16.5
else
  p result.message!
end
```

#### If you dont know what return type is, use MessagePack::Any:
```crystal
require "blink"

client = Blink::Client.new("127.0.0.1", 9000)
result = client.request!(MessagePack::Any, :bla, 3, "5.5")
p result.as_f + 1 # => 17.5
```

#### If you want to exchange complex data types, you should include MessagePack::Serializable to your data
```crystal
require "blink"

record MyResult, a : Int32, b : String { include MessagePack::Serializable }

class MyRequest
  include MessagePack::Serializable

  property a : Int32
  property b : Hash(String, String)?

  @[MessagePack::Field(ignore: true)]
  property c : Int32?
end

class MyRpc 
  include Blink::Protocol

  def doit(req : MyRequest) : MyResult
    # ...
  end
end
```

#### Example calling from Ruby, with gem msgpack-rpc
```ruby
require 'msgpack/rpc'

client = MessagePack::RPC::Client.new('127.0.0.1', 9000)
result = client.call(:bla, 3, "5.5")
p result # => 16.5
```
