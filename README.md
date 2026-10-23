# zconf

A layered configuration library for [Zell](https://github.com/zell-lang/zell),
modelled on the Go library [sconf](https://github.com/dvislobokov/sconf)
(itself modelled on `Microsoft.Extensions.Configuration`), without the Vault
part.

Every source is reduced to one flat model, `path -> string`, with `:` as the
separator (`servers:0:host`). Layers are merged in order and **the last one
wins per key**, so an environment variable can override a single field of
one array element defined in a file. Any struct binds from that model through
compile-time reflection (`reflect.Fields<T>()`), with no code generation.

What it shows about Zell:

- generic code that walks a struct at compile time (`reflect.Fields<T>()`,
  static `if reflect.TypeOf<T>().Kind == ...`) to bind nested structs,
  `[]struct`, `[]string`, `map[string]T`, `time.Duration`;
- a user-defined attribute (`@attribute struct Conf`) read both statically
  (binding) and at run time through `reflect.TypeInfo` (usage text);
- `T!` errors with `?`, `try ... catch`, sentinel errors wrapped with
  context and checked with `errors.Is`;
- interfaces (`Source`) and a chaining builder;
- `encoding.json`, `encoding.yaml`, `encoding.toml` dynamic values;
- `zell test` table of tests, one file per layer.

## Sources

| Builder method | Layer |
|---|---|
| `AddJsonFile(path, optional)` | JSON file |
| `AddYamlFile(path, optional)` | YAML file |
| `AddTomlFile(path, optional)` | TOML file |
| `AddDotEnvFile(path, prefix, optional)` | `.env` file, read like environment variables |
| `AddEnvironmentVariables(prefix)` | process environment, prefix stripped, `__` -> `:` |
| `AddEnvironment(prefix, ["K=V", ...])` | the same from explicit lines (tests) |
| `AddCommandLine(args)` | `--a:b=c`, `--a:b c`, bare `--flag` (= `true`), `--a__b=c` |
| `AddInMemory(map)` | in-memory map |
| `Add(source)` | any type implementing `zconf.Source` |

## Go vs Zell

<table>
<tr><th>Go (sconf)</th><th>Zell (zconf)</th></tr>
<tr><td>

```go
type Config struct {
    Host    string `default:"0.0.0.0"`
    Port    int    `default:"8080"`
    Mode    string `enum:"dev,prod" default:"dev"`
    Servers []struct {
        Host string
        Port int
    }
}

cfg, err := sconf.Load[Config](
    sconf.New().
        AddYAMLFile("appsettings.yaml",
            sconf.Optional()).
        AddEnvironmentVariables("APP_"),
    os.Args[1:],
)
if err != nil {
    log.Fatal(err)
}
```

</td><td>

```zell
struct Server {
    Host string
    Port int
}

struct Config {
    @Conf(default: "0.0.0.0") Host string
    @Conf(default: "8080") Port int
    @Conf(oneOf: "dev,prod", default: "dev") Mode string
    Servers []Server
}

func main() void! {
    var b = zconf.NewBuilder()
    b.AddYamlFile("appsettings.yaml", true).
        AddEnvironmentVariables("APP_")
    var cfg = zconf.Load<Config>(b, os.Args()[1..])?
    // ...
}
```

</td></tr>
</table>

Differences from sconf: Zell attributes are typed structs, so the tags become
one attribute `@zconf.Conf(name:, default:, oneOf:, required:, help:, skip:)`
(`enum` is a keyword in Zell, hence `oneOf`); the key name also honours
`@json(name:)`. Anonymous nested structs do not exist in Zell, so `Servers`
uses a named `Server` type.

## Binding rules

- Key name: `@Conf(name:)`, then `@json(name:)`, then the field name;
  matching is case-insensitive. Unexported fields and `@Conf(skip: true)`
  are skipped.
- Nested structs are sections (`database:host`); slices are indexed children
  (`servers:0:host`; gaps collapse) or one comma-separated value
  (`--tags=a,b`); maps are named children (`labels:team`).
- Missing keys keep the zero value, or `@Conf(default:)`. 
- Supported scalars: `string`, `int`, `float`, `bool`, `time.Duration`
  (`"15s"`).
- A later scalar replaces an earlier subtree at the same key and vice
  versa (`--tags=x,y` replaces `tags: [a, b]` from a file).

## Errors

All errors wrap a sentinel, so `errors.Is` works through the context:

| Sentinel | When |
|---|---|
| `zconf.ErrMissing` | `@Conf(required: true)` and no layer or default sets the key |
| `zconf.ErrEnum` | value outside `@Conf(oneOf:)` (`"mode" = "staging" (allowed: dev, prod)`) |
| `zconf.ErrParse` | value of the wrong type (`"servers:1:port" = "http" (expected an integer)`), malformed file |
| `zconf.ErrSource` | a required file is missing or unreadable |
| `zconf.ErrUnsupported` | a field type zconf cannot bind |

## `--help`

`Load` prints a usage generated from the struct and exits with status 0 when
the arguments contain `--help`, `-h`, `-?` or `/?` (`zconf.Usage<T>(prefix)`
returns the same text, `zconf.Describe<T>(prefix)` the entries):

```text
$ zell run -- --help
Options:
  --Name            string    (default "app")  [env APP_NAME]  service name
  --Level           string    {debug|info|warn|error}  (default "info")  [env APP_LEVEL]  log level
  --Timeout         Duration  (default "30s")  [env APP_TIMEOUT]  request timeout
  --Database:Host   string    (default "localhost")  [env APP_DATABASE__HOST]  database host
  --Database:Port   int       (default "5432")  [env APP_DATABASE__PORT]  database port
  --Servers:N:Host  string    [env APP_SERVERS__N__HOST]
  --Servers:N:Port  int       [env APP_SERVERS__N__PORT]

Built-in flags:
  --help, -h, -?  print this help and exit
```

## Ad-hoc access

```zell
var cfg = zconf.NewBuilder().AddYamlFile("appsettings.yaml", false).Build()?
cfg.GetString("database:host")
cfg.GetInt("database:port", 5432)
cfg.GetBool("database:ssl", false)
cfg.GetChildren("servers")          // ["0", "1"]
cfg.Section("database").GetString("host")
print(cfg.Dump())                   // "key = value" lines
```

## Install Zell

```sh
curl -fsSL https://raw.githubusercontent.com/zell-lang/zell/master/scripts/install.sh | sh
```

## Use it

```toml
# zell.toml
[dependencies]
zconf = { git = "github.com/zell-lang/zconf" }
```

## Run the example

```sh
cd example
zell run                                        # appsettings.yaml only
APP_SERVERS__0__HOST=env-host zell run -- --level=WARN --database:port 6543
zell run -- --level=trace                       # ErrEnum, exit status 1
zell run -- --help
```

```text
$ APP_SERVERS__0__HOST=env-host zell run -- --level=WARN --database:port 6543
name:     my-service
level:    warn
timeout:  15s
database: localhost:6543
server 0: env-host:8080
server 1: web2:8081
```

## Test

```sh
zell test        # 22 tests: flat model, every source, merge order, binding, errors, usage
```

## Limitations

- The Zell standard library has no `os.Environ()` yet, so
  `AddEnvironmentVariables` reads `/proc/self/environ` (Linux; elsewhere it
  sees no variables, and variables set with `os.Setenv` after start are not
  seen). `AddEnvironment(prefix, lines)` works everywhere.
- No Vault, no `Wait` for files, no `Dump` to JSON/YAML/TOML, no
  `--config-sample`, no `Unmarshaler`/`Validator` hooks yet.

## License

MIT
