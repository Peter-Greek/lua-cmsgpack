# luamsgpack-c
A [msgpack-c](https://github.com/msgpack/msgpack-c/tree/master) binding for Lua 5.1, Lua 5.2, Lua 5.3, Lua 5.4, and [LuaJIT](https://github.com/LuaJIT/LuaJIT) with the intention of being an API compatible replacement of [lua-MessagePack](https://fperrad.frama.io/lua-MessagePack/).

## Documentation
The exported API is broken down into five categories: **Configuration**, **Packing**, **Unpacking**, **Extensions**, and **Compatibility**. See **Developer Notes** for implementation details/caveats.

##### Configuration
```lua
-- Return the current value of the global packing/unpacking option.
--
-- Default Flags:
-- 'empty_table_as_array' + 'unsigned' + 'without_hole + 'double'
--
-- Options:
--  NUMBER_OPTS: See Developer Notes for how floating-point/integer types are
--  handled under the varying Lua builds/versions.
--   'float' - Encodes a lua_Number as float regardless of LUA_FLOAT_TYPE.
--   'double' - Encodes a lua_Number as double regardless of LUA_FLOAT_TYPE.
--   'integer' - Encodes a lua_Number as an integer regardless of type.
--   'unsigned' - Encode integers as unsigned values when possible, i.e., positive
--      lua_Integers are msgpacked as unsigned int; this is default for
--      lua-MessagePack.
--
--  STRINGS_OPTS: When both disabled the 'v5' spec for encoding strings is used.
--   'string_compat' - Use MessagePack 'v4' spec for encoding strings.
--   'string_binary' - Encode strings using the binary tag.
--
--  TABLE_OPTS:
--   'always_as_map' - Encode all tables as a sequence of <key, value> pairs.
--   'without_hole' - Only contiguous arrays, i.e., integer keys [1, N] all
--      contain non-nil value, are to be packed as an msgpack array type.
--   'with_hole' - Allow tables to be packed as arrays iff all keys are positive
--      integers and satisfies the MP_TABLE_CUTOFF limitation. Inserting nil's
--      when encoding to satisfy the array type.
--   'empty_table_as_array' - empty tables packed as arrays. Beware, when
--      'always_as_map' is enabled, this flag is forced to disabled.
--   'sentinel' - Replace 'nil' values with a 'sentinel' value during unpacking.
--      The packer will always replace sentinel's with null during packing.
--
--  TYPE_OPTS:
--   'ignore_invalid' - Ignore invalid types (i.e., ones without 'type' extensions)
--      during encoding by packing 'nil' (short circuiting) instead of throwing
--      an error.
--
--  COMPAT_OPTS:
--   'small_lua' - lua-MessagePack compatibility field.
--   'full64bits' - lua-MessagePack compatibility field.
--   'long_double' - lua-MessagePack compatibility field.
value = msgpack.getoption(option)

-- Set a global packing/unpacking option; see getoption.
msgpack.setoption(option, value)

-- Returns a sentinel value used to represent "null" arrays. Lua 5.1 and LuaJIT
-- require invoking the function while other Lua versions treat sentinel as a
-- 'light' C function, where msgpack.sentinel == msgpack.sentinel().
null = msgpack.null() -- or msgpack.sentinel()
```

##### Packing
```lua
-- Receives any number of arguments and pack their values.
packedString = msgpack.pack(...)

-- Returns a userdata that maintains a persistent msgpack packing state. The
-- userdata has the following metamethods
--
-- __len: Return the length of the current msgpack encoded string.
-- __tostring: Return the current msgpack encoded string.
-- __concat: Append another msgpack encoded strings to the packer.
-- __call, __add, __shl(>= 5.3): Encode, and append, the provided Lua values.
-- __index: Indexes functions of the form: f(packer, [, value [, ... [, value]...]])
--      Where the values are casted to the named type:
--      "nil",
--       "any",
--       "boolean", "true", "false",
--       "fix_uint8", "fix_uint16", "fix_uint32", "fix_uint64",
--       "fix_int8", "fix_int16", "fix_int32", "fix_int64",
--       "uint8", "uint16", "uint32", "uint64",
--       "int8", "int16", "int32", "int64",
--       "char", "signed_char", "unsigned_char",
--       "short", "integer", "long", "long_long",
--       "unsigned_short", "unsigned_int", "unsigned_long", "unsigned_long_long",
--       "signed_int16", "signed_int32", "signed_int64",
--       "integer", "signed", "unsigned",
--       "float", "double", "number",
--       "_string", "string_compat", "string", "binary",
--       "_table", "map", "array", "table"
--
-- @EXAMPLE:
--  ud = msgpack.new()
--  ud(1, 2, math.pi) -- Append; current state: { 1, 2, math.pi }.
--  ud .. tostring(ud) -- Duplicate; current state: { 1, 2, math.pi, 1, 2, math.pi }.
--  ud:float(4.0) -- Append; current state: { 1, 2, math.pi, 1, 2, math.pi, 4.0f }.
--  msgpack.unpack(tostring(ud)) -- Unpacks the current msgpack stream
packer = msgpack.new()
```

##### Unpacking
```lua
-- Unpack all elements, up to a potential limit, from a msgpack encoded string.
-- Returning the number of unpacked objects placed onto the Lua stack.
... = msgpack.unpack(encoded_string [, offset [, limit [, end_position]]])

-- MessagePack.lua ABI compatible unpack: ignore additional function arguments.
-- When compiled with LUA_MSGPACK_COMPAT, "unpack_compat" becomes the "unpack"
-- function, while "unpack" becomes "unpack2".
... = msgpack.unpack_compat(encoded_string)

-- Unpack all elements, up to a potential limit, from a msgpack encoded string.
-- Returning (1) the position in the string where the decoding ended, 0 for
-- completion; and (2) and all decoded objects (up to limit).
--
-- Iterator Example:
--      local position,element = 1,nil
--      while position ~= 0 do
--        position,element = msgpack.next(encoded_string, position, 1)
--      end
new_position,... = msgpack.next(encoded_string [, position [, limit [, end_position ]]])
```

##### Extensions
```lua
-- Register an extension-type. The encoder_table parameter is often a metatable
-- with additional metamethods for serializing tables/userdata:
--
--   __ext: Unique msgpack extension identifier. Note that applications can only
--    assign 0 to 127 to store application-specific type information.
--
--   __pack: An encoder function:
--              encoding[, handled_header] = f(self, type)
--    where "type" is the extension type identifier (allowing one function
--    handling multiple encodings). The 'handled_header' result is an optional
--    field to tell the encoder that the serialization function 'f' handled
--    packing its extension header.
--
--   __unpack: A decoder function: value = f(encoded_string) that is the inverse
--    to __pack. Note, 'f' may only return one value as the custom extension
--    types may be used to encode key/values in tables/arrays.
--
-- @EXAMPLE:
--   metatable = {
--     __ext = 0x15,  -- Extension type identifier
--
--     __pack = function(self, type) -- Object Serialization
--          return msgpack.pack(self.x, self.y, self.z)
--     end,
--
--     __unpack = function(encoded, type) -- Factory
--          local x,y,z = msgpack.unpack(encoded)
--          return setmetatable({x = x, y = y, z = z}, metatable)
--     end,
--
--    --[[ Metamethods --]]
--  }
msgpack.extend(encoder_table)

-- Get the extension-type definition for encoding/decoding tables/userdata
-- definitions.
metatable = msgpack.extend_get(ext_id)

-- Explicitly remove the extension definition for all type identifiers provided
-- to this function; returning zero.
msgpack.extend_clear(ext_id1 [, ext_id2 ... [, ext_idN]])

-- Associate the name of a Lua type (see: lua_typename/type) to a encoder table,
-- and possibly a unique MessagePack extension type identifier.
--
-- @NOTE: This feature is going to be reworked.
-- @EXAMPLE:
--  m.extend("function", {
--      __ext = 42,
--
--      __pack = function(fct, t)
--          assert(type(fct) == "function", "is function")
--          return m.pack(assert(string.dump(fct), "function pack"))
--      end,
--
--      __unpack = function(s, t)
--          local str = m.unpack(s)
--          return assert(loadstring(str), "function unpack")
--      end,
--  })
--
-- @EXAMPLE:
--  m.extend("function", 42) -- '42' is an already registered extension identifier
msgpack.settype(type_string [, ext_id])

-- Get the encoder table associated to the name of a Lua type.
msgpack.gettype(type_string)
```

##### Compatibility
```lua
-- lua-MessagePack:
-- setoption that only processes: "string", "string_compat", "string_binary"
msgpack.set_string(string_value)
-- setoption that only processes: "without_hole", "with_hole", "always_as_map"
msgpack.set_array(array_value)
-- setoption that only processes: "signed", "unsigned"
msgpack.set_integer(integer_value)
-- setoption that only processes: "float", "double"
msgpack.set_number(number_value)
```

## Building
A CMake project that builds the shared library is included. See `cmake -LAH` or [cmake-gui](https://cmake.org/runningcmake/) for the complete list of build options.

```bash
# Create build directory:
└> mkdir -p build ; cd build
└> cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release ..

# Using a custom Lua build (Unix). When using Windows, -DLUA_LIBRARIES= must also
# be defined for custom Lua paths. Otherwise, CMake will default to 'FindLua'
└> cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release -DLUA_INCLUDE_DIR=${LUA_DIR} ..

# Build
└> make
```

### Compile Options
- **LUA\_COMPILED\_AS\_HPP**: Library compiled for Lua with C++ linkage.
- **LUA\_INCLUDE\_TEST**: Build with `LUA_USER_H="ltests.h"`.
- **LUA\_MSGPACK\_COMPAT**: Strict lua-MessagePack compatibility.
- **LUA\_MSGPACK\_SAFE**: Wrap all API functions in pcall.
- **LUA\_MSGPACK\_BIT32**: i386 compilation. Uses the float/int32 msgpack-c functions for floating-point/integer types.
- **LUA\_MSGPACK\_ERROR\_NESTING**: Throw a `lua_error` when exceeding the maximum table nesting depth (often the consequence of recursive tables). Otherwise, short circuit the nesting by packing nil.
- **MP\_MAX\_NESTING**: Maximum recursive-depth/table-nesting.
- **MP\_TABLE\_CUTOFF**: Threshold for mp_table_is_an_array. If a table of only integer keys has a key greater than this value: ensure at least half of the keys within the table have non-nil objects to be encoded as an array. This is technically not one-to-one with MessagePack.lua.
- **MP\_BUFFER\_INITSIZE**: Initial intermediate string buffer size.
- **MP\_ZONE\_CHUNK\_SIZE**: Default chunk msgpack_zone chunk size.

## Developer Notes
1. Large values, e.g., uint64_t (`0xcf`) or float64 (`0xcb`), may not be able to be represented in Lua (especially when compiled for i386 or C89).
1. For Lua 5.1, Lua 5.2, and LuaJIT, all unpacked integers are type-casted to floating point types (see the C and/or C++ standard for type-casting rules). A `LUA_TNUMBER` will be packed as an integer type if the value can be faithfully represented as an integer, i.e., `(lua_Number)floor(value) = value`.
1. For Lua 5.3 and Lua 5.4 a `LUA_TNUMBER` value will be packed as an integer if `lua_isinteger` returns true for the given value. For default [PUC-Rio Lua](https://github.com/lua/lua) this requires the value to have an explicit integer type.

### TODO
1. An actual C API.
1. `zone.c` uses `malloc/realloc` and does not support custom allocators. Introduce a zone implementation that uses lua_Alloc.
1. A `clear` function for `msgpack.new`, allowing its internal string buffer to be reset.
1. `pack`: experiment with an additional table parameter that can be used to cache already processed tables. The current solution relies on maximum recursive depth while being incredibly defensive around the state/size of the Lua stack.
1. Replace 'next' with something more efficient, e.g, a `msgpack.iterator` persistent userdata. The current iterator approach is incredibly inefficient as its continuously creating and destroying msgpack_zones and whatever Lua overhead to ensure no leakage.

## Sources & Acknowledgments:
1. [msgpack-c](https://github.com/msgpack/msgpack-c): msgpack spec implementation;
1. [antirez/lua-cmsgpack](https://github.com/antirez/lua-cmsgpack): original implementation. Written by Salvatore Sanfilippo for Redis, but maintained as a separated project "in a self contained C file without external dependencies";
1. [lua-MessagePack](https://fperrad.frama.io/lua-MessagePack): API compatibility reference;
1. [MessagePack-JS](https://github.com/cuzic/MessagePack-JS): Some test cases in [test.lua](test/test.lua) are obtained from the MessagePack-JS library;

## License
luamsgpack-c is distributed under the terms of the [MIT license](https://opensource.org/licenses/mit-license.html); see [lua_cmsgpacklib.h](src/lua_cmsgpacklib.h)
