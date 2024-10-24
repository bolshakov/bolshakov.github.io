---
layout: post
title: "Introducing ConfigX: A Simple, Yet Flexible Ruby Configuration Library"
date: 2024-10-24 00:00:00 -0400
---

Imagine you're scaling your Ruby application across multiple environments - development, staging, and production. Each 
environment demands its own configuration, and you need to ensure sensitive data stays secure. As your application 
grows, managing these configurations becomes increasingly complex. How do you handle this efficiently without 
compromising on flexibility or security?

Enter [ConfigX] - a lightweight, yet powerful Ruby configuration library designed to simplify this exact challenge. 
Whether you're building a small service or a complex application, **ConfigX** offers a clean, structured approach to 
configuration management that grows with your needs. In this post, we'll explore how **ConfigX** can improve the way 
you handle configuration in Ruby projects, making your development process smoother.

## What is ConfigX?

**ConfigX** is built with simplicity and flexibility in mind. Unlike other configuration libraries, **ConfigX** allows 
you to define multiple independent configurations, each loaded into its own Ruby object. This approach avoids creating 
global objects, making it easier to manage different configuration sources separately, especially in complex applications, 
without the risk of unwanted coupling.

## Key Features and Usage Examples

### 1. Multiple Configuration Sources

**ConfigX** merges configuration from YAML files and environment variables, giving you flexibility in how you manage 
your settings.

```yaml
# config/settings.yml
database:
  host: localhost
  port: 5432
```

```ruby
ENV['SETTINGS__DATABASE__HOST'] = 'production.db.example.com'

config = ConfigX.load
puts config.database.host  # Output: production.db.example.com
puts config.database.port  # Output: 5432
```

This feature allows you to easily override default settings with environment-specific values, crucial for managing 
different deployment environments.

### 2. Environment Variable Parsing

**ConfigX** automatically converts environment variables into appropriate data types, saving you from manual type 
conversion and reducing potential errors.

```ruby
ENV['SETTINGS__DEBUG'] = 'true'
ENV['SETTINGS__MAX_CONNECTIONS'] = '5'
ENV['SETTINGS__ALLOWED_IPS'] = '["192.168.1.1","192.168.1.2"]'

config = ConfigX.load
puts config.debug.class             # Output: TrueClass
puts config.max_connections.class   # Output: Integer
puts config.allowed_ips.class       # Output: Array
```

This feature is particularly useful when working with containerized applications where configuration via environment 
variables is common.

### 3. Typed Configuration

Leveraging [dry-struct], **ConfigX** allows you to define types and validation for your configuration, catching errors 
early in the development process.

```ruby
class AppConfig < ConfigX::Config
  attribute :database do
    attribute :host, Types::String
    attribute :port, Types::Integer
    attribute :max_connections, Types::Integer.default(5)
  end
end

config = ConfigX.load(config_class: AppConfig)
puts config.database.host
puts config.database.max_connections
```

This type-checking ensures that your configuration values match expected types.

### 4. No Global Configuration

Each configuration instance in **ConfigX** is isolated, allowing you to manage multiple configurations without conflicts.

```ruby
web_config = ConfigX.load(file_name: 'web_settings')
job_config = ConfigX.load(file_name: 'job_settings')

puts web_config.server.port
puts job_config.queue.names
```

This feature is particularly useful in complex applications where different components may require different 
configuration setups.

### 5. Environment-Aware Configuration

**ConfigX** automatically loads the appropriate configuration based on the current environment, simplifying management 
of environment-specific settings.

## Deep Dive: Environment-Aware and Local Configuration Files

One of **ConfigX**'s most powerful features is its sophisticated handling of environment-specific and local configuration 
files. This system provides a flexible way to manage settings across different environments and allows for local 
overrides, which is important for team development and secure handling of sensitive information.

### File Loading Order

**ConfigX** loads configuration files in a specific order, with each subsequent file overriding values from the previous ones:

1. Base configuration file (`config/settings.yml`)
2. Environment-specific configuration file (`config/settings/{environment}.yml`)
3. Local base configuration file (`config/settings.local.yml`)
4. Local environment-specific configuration file (`config/settings/{environment}.local.yml`)
5. Environment variables

### File Naming Conventions

**ConfigX** uses a consistent naming convention:

- Base configuration: `config/settings.yml`
- Environment-specific: `config/settings/{environment}.yml`
- Local overrides:
    - `config/settings.local.yml`
    - `config/settings/{environment}.local.yml`

Where `{environment}` can be `development`, `test`, `production`, or any custom environment name you define.

### Example Scenario

Let's walk through a practical example:

```yaml
# config/settings.yml (Base configuration)
database:
  port: 5432
log_level: info

# config/settings/development.yml (Development environment)
database:
  host: localhost

# config/settings.local.yml (Local override, not in version control)
log_level: debug

# config/settings/development.local.yml (Local development override, not in version control)
database:
  port: 5433
```

When loaded in the development environment, the resulting configuration would be:

```ruby
config = ConfigX.load('development')

puts config.database.host  # Output: localhost
puts config.database.port  # Output: 5433
puts config.log_level      # Output: debug
```

This system allows you to:
- Manage different settings for various environments
- Allow developers to have their own local settings
- Keep sensitive data in local files outside of version control
- Use as many or as few of these files as needed for your project

### Version Control Considerations

To keep sensitive or personal configurations out of version control, you can add the following to your `.gitignore`:

```
config/settings.local.yml
config/settings/*.local.yml
```

This ensures that the base and environment-specific configurations are shared with the team, while local overrides 
remain on individual machines.

## Customizing Configuration

**ConfigX** allows you to customize how configurations are loaded:

```ruby
ConfigX.load(
  "development",
  dir_name: 'settings',
  file_name: 'settings',
  config_root: 'config',
  env_prefix: 'SETTINGS',
  env_separator: '__'
)
```

This flexibility allows you to adapt **ConfigX** to your project's specific needs and conventions.

## Getting Started with ConfigX

Ready to simplify your configuration management? Here's how to get started:

1. Add ConfigX to your Gemfile:
```ruby
gem 'configx'
```

2. Run bundle install:
```
bundle install
```

3. Create your configuration files following the naming conventions described above.

4. Load and use your configuration in your application:
```ruby
# config/application.rb or similar
environment = ENV['RAILS_ENV'] || 'development'
AppConfig = ConfigX.load(environment)
```

## ConfigX vs Other Configuration Libraries

Let's compare **ConfigX** with two popular Ruby configuration libraries: [dry-configurable] and [config].

### ConfigX vs dry-configurable

| Feature | ConfigX                                    | dry-configurable |
|---------|--------------------------------------------|------------------|
| Focus | Loading settings from multiple sources     | Object-specific configurations |
| Use case | Straightforward, centralized configuration | Highly modular applications |
| External file loading | Built-in support for YAML files            | Requires custom implementation |
| Environment variable support | Built-in, with automatic parsing           | Requires custom implementation |
| Local overrides | Supported out of the box                   | Not directly supported |

[dry-configurable] is part of the [dry-rb] ecosystem and excels in highly modular or domain-driven applications where 
each class can define its own configuration. **ConfigX**, on the other hand, is simpler and better suited for 
applications that need a centralized configuration that can be loaded from external sources.

If you use **dry-configurable** you can still benefit from **ConfigX**, by loading configuration from different sources
and then configuring your classes with the loaded values.

### ConfigX vs config

| Feature | ConfigX | config                      |
|---------|---------|-----------------------------|
| Global state | Avoids global objects | Uses global configuration   |
| Multiple configurations | Supports multiple independent configs | Single global configuration |
| Typed configuration | Supports via **dry-struct** | Not supported               |

The [config] gem is one of the most popular configuration libraries for Ruby, known for its ability to load 
configuration from YAML files and manage environment-specific settings. However, it uses a global configuration 
object, which can be limiting in more complex scenarios. **ConfigX** avoids global objects, allowing for more 
flexibility in managing multiple independent configurations.

## Why Choose ConfigX?

ConfigX offers a fresh approach to Ruby configuration management with its focus on simplicity, flexibility, and
type safety. It allows you to:

- Keep your configuration organized and type-safe
- Easily manage different environments without code changes
- Avoid global state and potential conflicts in complex applications
- Catch configuration errors early in the development process
- Securely manage sensitive information through local overrides
- Customize configuration loading to fit your project's needs

Whether you're building a small application or scaling a complex system, **ConfigX** provides the tools you need to 
handle configuration with confidence. Ready to transform your configuration management and make your Ruby applications 
more robust? Give **ConfigX** a try in your next project! Check out [ConfigX on GitHub] to get started.


[dry-configurable]: http://dry-rb.org/gems/dry-configurable
[config]: https://github.com/rubyconfig/config
[ConfigX]: https://github.com/bolshakov/configx
[dry-rb]: https://dry-rb.org
[dry-struct]: https://dry-rb.org/gems/dry-struct/
[ConfigX on GitHub]: https://github.com/bolshakov/configx
