## Introduction

Iroha is configurable. Configuration is almost the first thing each Iroha user faces, and it is important for this part to be as reliable as possible.

This RFC aims to make Iroha configuration reliable. (and more functional? more maintainable?)

## Background

Currently Iroha is configurable with `config.json` file and a list of environment variables.

There are several issues with the way Iroha configuration is currently defined:

- unconfigurable fields being exposed to the user
- the use of rustisms in configuration reference
- ambiguity in setting root-level fields
- the use of redundant SCREAMING_CASE in JSON configuration
- overcomplicated way of configuration via ENV variables
- inconsistent naming of ENV variables
- unhelpful error messages
- chaotic code organisation

Let's take a closer look at each of these issues.

### Redundant fields

There are configuration fields that shouldn't be initialized by the user, meaning that including them in the configuration reference is redundant. For example, here's the excerpt about configuration options for Sumeragi:

> **Sumeragi: default `null` values**
>
> A special note about sumeragi fields with `null` as default: only the `trusted_peers` field out of the three can be initialized via a provided file or an environment variable.
>
> The other two fields, namely `key_pair` and `peer_id`, go through a process of finalization where their values are derived from the corresponding ones in the uppermost Iroha config (using its `public_key` and `private_key` fields) or the Torii config (via its `p2p_addr`). This ensures that these linked fields stay in sync, and prevents the programmer error when different values are provided to these field pairs. Providing either `sumeragi.key_pair` or `sumeragi.peer_id` by hand will result in an error, as it should never be done directly.

As we can see, if the user tries to initialize `key_pair` and `peer_id` via the config file or env variables, they'll get an error. We are creating a terrible user experience by presenting the user with an option to configure something they should not be configuring.

We should avoid exposing fields like this to the user.

### Rustisms in the configuration reference

Configuration reference contains Rust-specific expressions that wouldn't make sense to the end-user who is configuring Iroha and might not be familiar with Rust. Here are some examples:

- [Using `Option<..>`](https://github.com/hyperledger/iroha/blob/35ba182e1d1b6b594712cf63bf448a2edefcf2cd/docs/source/references/config.md#option)
- [Using `std::path::PathBuf`](https://github.com/hyperledger/iroha/blob/35ba182e1d1b6b594712cf63bf448a2edefcf2cd/docs/source/references/config.md#loggerlog_file_path)

The use of rustisms like that overcomplicates the configuration reference and affects the user experience.

### Ambiguity in setting root-level fields

To understand the issue with ambiguity at the root-level fields, let's once again look at the excerpt from the [configuration reference](https://github.com/hyperledger/iroha/blob/35ba182e1d1b6b594712cf63bf448a2edefcf2cd/docs/source/references/config.md#network):

> **`network`**
>
> Network configuration
>
> Has type `Option<network::ConfigurationProxy>`. Can be configured via environment variable `IROHA_NETWORK`
>
> ```json
> {
>   "ACTOR_CHANNEL_CAPACITY": 100
> }
> ```

As we can see, `network` is not actually a configuration field in itself. Instead, there are various `network.*` fields that can be configured. This bring a question of what will happen when we set the `IROHA_NETWORK` environment variable as mentioned in the excerpt above? Will it override the nested `IROHA_NETWORK_CAPACITY` fields? Is there a use case for providing an ability to set `IROHA_NETWORK` all-in-one through an environment variable?

### Redundant SCREAMING_CASE in JSON configuration

This is how JSON configuration looks like:

```json
{
  "PUBLIC_KEY": null,
  "PRIVATE_KEY": null,
  "DISABLE_PANIC_TERMINAL_COLORS": false
}
```

There is no clear reason why keys should be uppercase.

### Overcomplicated ENV variables

TODO. See "Sane aliases" section below

### Inconsistent naming of ENV variables

If we look at the names of environment variables, there are some variables that use the `IROHA2` prefix, while others use `IROHA`. Check the output of the `./iroha --help` command:

```
Iroha 2
pass `--help` (short `-h`) for this message
pass `--submit-genesis` (short `-s`) to submit genesis from this peer
pass `--version` (short `-V`) to print version information

Iroha 2 is configured via environment variables:
    IROHA2_CONFIG_PATH is the location of your `config.json` or `config.json5`
    IROHA2_GENESIS_PATH is the location of your `genesis.json` or `genesis.json5`
If either of these is not provided, Iroha checks the current directory.

Additionally, in case of absence of both IROHA2_CONFIG_PATH and `config.json` or `config.json5`
in the current directory, all the variables from `config.json` or `config.json5` should be set via the environment
as follows:
    IROHA_TORII: Torii (gateway) endpoint configuration
    IROHA_SUMERAGI: Sumeragi (emperor) consensus configuration
    IROHA_KURA: Kura (storage). Configuration of block storage
    IROHA_BLOCK_SYNC: Block synchronisation configuration
    IROHA_PUBLIC_KEY: Peer public key
    IROHA_PRIVATE_KEY: Peer private key
    IROHA_GENESIS: Genesis block configuration
Examples of these variables can be found in the default `configs/peer/config.json`.
```

Also, `IROHA2_CONFIG_PATH` doc says "specify location to your `config.json`", which is not correct. You can specify location to _any_ file, with any name and extension (e.g. `my_config.json`, `config.foobar`, `cfg` etc), but it should be parseable as an UTF-8 JSON-encoded string.

### Unhelpful error messages

Some of the error messages related to Iroha configuration are inconsistent and misleading. Here's a quick rundown of various scenarios, associated errors and possible confusions.

#### 1. No configuration file

Running Iroha without a configuration file results in the following log and error:

```
IROHA_TORII: environment variable not found
IROHA_SUMERAGI: environment variable not found
IROHA_KURA: environment variable not found
IROHA_BLOCK_SYNC: environment variable not found
IROHA_PUBLIC_KEY: environment variable not found
IROHA_PRIVATE_KEY: environment variable not found
IROHA_GENESIS: environment variable not found
Configuration file not found. Using environment variables as fallback.
Error:
   0: Please add `PUBLIC_KEY and PRIVATE_KEY` to the configuration.
```

Several things are wrong with this error message:

- The information about using environment variables as fallback comes after the messages about them not being found.
- Hinting to add `PUBLIC_KEY` and `PRIVATE_KEY` to the configuration does not explain what happened (the absence of the config file and no environment variables to fallback to). This hint is also not helpful as there is no information on whether public and private keys should be added to the config file or to ENV variables, and no information about the ENV variables these fields are mapped to.

#### 2. Path to non-existent config file

Providing a path to a configuration file that doesn't exist results in the exact same error as if there was no configuration file specified at all.

While it makes sense to silently fallback to ENV variables if the user doesn't provide a path to a config file, in case when the user **does** specify the path to a config file, it would be better for the program to fail with an appropriate error message.

#### 3. Empty config file

Running Iroha with an empty `config.json` results in the error that does not provide any information about the fallback to the environment variables even though it does happen. This behaviour is inconsistent.

```
Error:
   0: Please add `PUBLIC_KEY and PRIVATE_KEY` to the configuration.
```

#### 4. Config file with only `PUBLIC_KEY` and `PRIVATE_KEY` specificed

Running Iroha with a config file that only contains `PUBLIC_KEY` and `PRIVATE_KEY` results in the following error:

```
Error:
   0: Please add `p2p_addr` to the configuration.
```

This error message is misleading and inconsistent:

- If the `p2p_addr` field is added to the config file, the error stays the same.
- The `p2p_addr` field is not a root-level field, it's actually a part of the `TORII` configuration and should be `TORII.P2P_ADDR`.
- Error messages asking to add public and private keys use uppercase for configuration fields, while in this error the field name is written in lowercase.

#### 5. Config file with only `PUBLIC_KEY`, `PRIVATE_KEY`, and `TORII.P2P_ADDR` specified

Running Iroha with a config file that only contains `PUBLIC_KEY`, `PRIVATE_KEY`, and `TORII.P2P_ADDR` results in the following error:

```
Error:
   0: Please add `api_url` to the configuration.
```

Comparing this scenario to the previous one, we can notice that the `TORII.API_URL` was missing before as well but the previous error message didn't mention it.

There are also two inconsistencies:

- Similar to `p2p_addr` in the previous scenario, this field name is lowercase, while for public and private keys it was uppercase.
- While `TORII.P2P_ADDR` and `TORII.API_URL` share the same root-level (`TORII`), their names are different: "addr" derived from "address" and "URL". Logically these are both addresses or URLs, and it would make sense for the names to be aligned.

#### 6. Config file with invalid `PUBLIC_KEY`

Providing an invalid `PUBLIC_KEY` in the `config.json` results in the same error as when there is no config file at all, or an empty config file:

```
Error:
   0: Please add `PUBLIC_KEY and PRIVATE_KEY` to the configuration.
```

This is not a helpful error message as it does not mention what exactly is wrong, why and where the parsing failed.

#### 7. Invalid `IROHA2_PUBLIC_KEY` environment variable

Providing an invalid `IROHA2_PUBLIC_KEY` environment variable results in the following error:

```
Error:
   0: Failed to build configuration from env
   1: Failed to deserialize the field `IROHA_PUBLIC_KEY`: JSON5: Key could not be parsed. Key could not be parsed. Invalid character 's' at position 4
   2: JSON5: Key could not be parsed. Key could not be parsed. Invalid character 's' at position 4
```

While this message is more helful than the ones we discussed above, there are still issues:

- The message contains repetition.
- Without an input snippet, the `Invalid character 's' at position 4` part of the message is not as helpful as it could be.

#### 8. Config file with extra fields

Providing a configuration file with extra fields that are not supposed to be there does not produce any errors at all.

This might lead to a bad user experience when user expects some options to apply, but doesn't have any idea that those in fact are silently ignored.

#### 9. Specifying `SUMERAGI.PUBLIC_KEY`

As you can see in the "Redundant fields" section, user should not specify any non-null value for `SUMERAGI.PUBLIC_KEY` parameter. However, if they do, they will get the following error:

```
Error:
   0: Please add `PUBLIC_KEY and PRIVATE_KEY` to the configuration.
```

The problem is that the error message tells something completely unrelated to the actual cause. This produces a terrible experience for users.

#### 10. Config file is an invalid JSON file

Providing an invalid JSON file as a config results in the same error we've seen before, which is not useful at all in this particular case and doesn't tell the user anything about the invalid JSON:

```
Error:
   0: Please add `PUBLIC_KEY and PRIVATE_KEY` to the configuration.
```

### Chaotic code organisation

Internally, not all of the configuration-related logic is contained within a single `iroha_config` crate. Instead, some of the configuration resolution logic is located in other crates, such as `iroha_cli`. This makes the configuration-related code error-prone and harder to maintain.

## List of Points

Having identified the issues with Iroha configuration, we now propose the following improvements:

1. Introduce exhaustive error messages
2. Create the reference before the implementation
3. Use better aliases for configuration parameters
4. Define deprecation and migration policy
5. Use TOML
6. Trace configuration resolution
7. Introduce two-stage configuration resolution
8. Introduce consistent naming
   ...

_TODO: update the list_

Let's take a closer look at each of these points.

### 1. Introduce exhaustive error messages

This proposal advocates for the implementation of exhaustive error messages in Iroha during the configuration stage. The objective is to enhance user troubleshooting by providing comprehensive and informative error messages for different types of configuration errors.

#### Objective

The objective of this proposal is to improve user troubleshooting during the configuration stage by introducing exhaustive error messages for two types of errors that users might encounter.

#### Rationale

Configuration errors can be challenging for users to diagnose and resolve without detailed information. By providing comprehensive error messages, we address the following key points:

1. **Clarity in Parsing Errors:**
   - **Location of Invalid Data:** By indicating the location of invalid data in the configuration source, users can easily identify problematic sections and rectify them promptly.
   - **Missing Fields Information:** Providing a list of missing fields allows users to quickly determine which essential information is absent and needs to be added to the configuration.
   - **Handling Unknown Fields:** Informing users about unknown provided fields helps them spot typos or irrelevant data that might have been inadvertently included.
2. **Guidance for Resolving Configuration:**
   - **Iroha-Specific Details Consideration:** When resolving the final configuration, it is essential to take all higher-level domain-specific (Iroha-specific) details into account. Informing users about any resolution failures helps them address potential conflicts and ensures a properly resolved configuration.
3. **Avoiding Silent Ignorance:**
   - **Transparent Error Reporting:** By avoiding silent ignorance of invalid data, we improve the transparency of the configuration system. Users will receive clear feedback on any configuration errors, allowing them to take immediate action.
4. **User-Friendly Error Messages:**
   - **Informative and Synchronized Messages:** The proposed error messages should be informative and follow a consistent style across the project. Synchronized messages help maintain consistency and improve user understanding.
5. **Handling Multiple Missing Fields Efficiently:**
   - **Listing Multiple Missing Fields:** When more than one field is missing, listing all of them at once instead of one-by-one provides a better user experience and streamlines troubleshooting.

#### Summary

By providing exhaustive error messages, Iroha can significantly improve user troubleshooting during the configuration stage, providing users with clear guidance to address configuration-related issues effectively.

**Implementation plan:** TODO include example errors

### 2. Create the reference before the implementation

The proposal suggests an approach that focuses on creating a clear and comprehensive configuration reference before implementing the configuration logic in code. This method aims to improve user experience and code maintainability by using domain-specific language in the reference and separating it from developer-specific concepts.

#### Objective

The objective of this proposal is two-fold:

- **To create a user-friendly configuration reference:** By separating the user and developer references and purging non-domain-specific concepts like `Option`, `enum`, `null`, and other Rust-specific language, the resulting configuration reference becomes more accessible and intuitive for end users. This approach ensures that users are presented with relevant configuration options and parameters, making the overall configuration process more straightforward and user-friendly.
- **To improve implementation quality and maintainability:** By using domain-specific language in the configuration reference, the implementation of configuration logic can be better aligned with user expectations. Writing tests based on this user-focused reference provides clear specifications for the implementation. It ensures that the code behaves as intended from a user's perspective, enhancing the overall quality and maintainability of the codebase.

#### Rationale

The rationale behind this proposal lies in the benefits it brings to both end users and developers:

- **Enhanced User Experience:** By eliminating non-domain-specific concepts from the configuration reference, we can provide end users with a concise and relevant set of configuration options. This enhances the user experience, reducing confusion and making the application easier to configure.
- **Improved Code Quality:** Writing tests based on the user-friendly configuration reference acts as a strong contract between the reference and the implementation. It ensures that the configuration logic is aligned with the intended behavior described in the reference, leading to fewer bugs and better code quality.
- **Streamlined Maintenance:** Separating the user and developer references allows developers to focus on the implementation without worrying about exposing internal language details to end users. This separation simplifies maintenance efforts and allows for more flexible code changes without affecting user-facing aspects.
- **Efficient Refactoring:** With a clear and user-focused reference, developers can confidently refactor the configuration logic, knowing that any changes are guided by the reference's specifications. This results in a more agile and maintainable codebase.

#### Summary

By implementing this approach, we can significantly improve the user experience, code quality, and overall maintainability of the software project's configuration system.

### 3. Use better aliases for configuration parameters

This proposal suggests using more user-friendly aliases for configuration parameters in Iroha. The aim is to enhance accessibility and improve the overall user experience.

#### Objective

The objectives of this proposal are:

1. **User-Friendly Aliases:** Rename internal configuration parameters with generic names that align with user expectations, making the configuration system more accessible.
2. **Clear Trace Logs and Reference:** Ensure trace logs and the configuration reference consistently use these aliases for better user understanding.
3. **Shorthand Names for Environment Variables:** Provide shorthand names for environment variables to streamline the configuration process.

#### Rationale

The benefits of this proposal include:

- **Improved User Experience:** User-friendly aliases make the configuration system less intimidating and more intuitive for both new and experienced users.
- **Simplified Maintenance:** Clear and user-oriented aliases improve code readability and ease the development process.
- **Consistency and Clarity:** Using aliases consistently in trace logs and the configuration reference reduces confusion during troubleshooting and debugging.
- **Efficient Customization:** Shorthand names for environment variables enhance the configuration process for different environments.

#### Summary

By implementing this approach, the Iroha configuration system can become more accessible, user-friendly, and aligned with user expectations.

### 4. Introduce consistent naming

This proposal aims to introduce consistent naming in Iroha. By renaming specific fields, we can improve clarity and maintain a more uniform naming convention.

#### Objective

The objective of this proposal is to achieve better consistency in naming for improved code readability and maintainability.

#### Rationale

The rationale behind this proposal includes:

- **Clarity and Readability:** Consistent naming improves code clarity, making it easier for developers to understand the purpose of each field at a glance.
- **Maintainability:** A uniform naming convention simplifies maintenance efforts, as developers can expect similar names for related functionalities throughout the project.

#### Renamed Fields

1. `torii`:
   - Renamed from `p2p_addr` to `addr_p2p`.
   - Renamed from `api_url` to `addr_api`.
   - Renamed from `telemetry_url` to `addr_telemetry`.
2. `logger`:
   - Renamed from `max_log_level` to `log_level`, or even just `level`.
3. `wsv`:
   - Renamed from `wasm_runtime_config` to `wasm_runtime`.

TODO: maybe also move root `public_key` & `private_key` to `iroha.public_key` and `iroha.private_key`?

#### Summary

By implementing these consistent naming changes, Iroha can benefit from improved code clarity and maintainability.

### 5. Define deprecation and migration policy

This proposal aims to establish a clear deprecation and migration policy specifically for the configuration part of Iroha. The objective is to provide a structured approach to handle changes, deprecations, and migrations related to the project's configuration options.

#### Objective

The objective of this proposal is to ensure seamless configuration evolution by defining a deprecation and migration policy for the configuration part of Iroha.

#### Rationale

- **Configuration Evolution:** With the project's continuous development, certain configuration options may need to be updated, replaced, or deprecated. A well-defined policy ensures that these changes are managed efficiently and consistently.
- **User Communication:** By communicating configuration changes, deprecations, and migration paths effectively, users can adapt to updates smoothly and minimize disruptions.

#### Deprecation Policy

- **Criteria and Conditions:** Specify the criteria and conditions that warrant deprecating a configuration option in the project. For example, when an option becomes obsolete, is replaced by an improved alternative, or poses potential security risks.
- **Deprecation Process:** Define the process for marking configuration options as deprecated, including clear documentation updates and notifications to users.
- **Phasing Out Timeline:** Outline a timeline for phasing out deprecated configuration options, allowing users sufficient time for migration.

#### Migration Policy

- **Guidelines for Migration:** Provide guidelines and best practices for users to follow when migrating from deprecated configuration options to the recommended alternatives.
- **Documentation and Support:** Offer clear and comprehensive documentation to facilitate the migration process. Provide support and assistance to users during the migration phase.

#### Summary

_TODO: Detail the steps to implement the defined deprecation and migration policy, including updating the configuration documentation, codebase, and communication channels._

By defining a robust deprecation and migration policy for the configuration part of Iroha, the project can ensure smooth configuration evolution and maintain a positive user experience during transitions.

### 6. Use TOML

TOML is a preferred configuration format for Iroha due to its simplicity, machine-readability, and easy-to-understand configuration references. The adoption of TOML enhances the configuration system, making it more user-friendly and maintainable.

#### Objective

The objective of this proposal is to transition to TOML as the standard configuration format in Iroha. It includes using lowercase for parameters.

#### Rationale

- **TOML vs. JSON:** TOML offers several advantages over JSON, which is currently in use for configuration. These benefits include a more human-friendly syntax, support for comments, explicit date and time types, and improved readability, making it easier for developers to work with the configuration reference.
- **Machine-Readable and User-Friendly:** TOML's machine-readable nature enables better tooling support and automated processing, streamlining configuration management tasks. Its user-friendly syntax and readability encourage more contributors to engage with the configuration system.
- **Consistency and Standardization:** TOML is widely adopted in the Rust ecosystem and serves as the standard configuration format for Rust projects. As Iroha is written in Rust, using TOML ensures consistency across the project and aligns with best practices in the Rust community.
- **Get Rid of Redundant SCREAMING_CASE:** there is no clear reason why the fields are uppercase in the current configuration.

#### Summary

By transitioning to TOML, Iroha can benefit from a more user-friendly, consistent, and maintainable configuration system, while also leveraging TOML's advantages over JSON as the current configuration format.

### 7. Trace configuration resolution

To enhance transparency and provide users with clear insights into configuration resolution, the proposal suggests implementing a trace mechanism for configuration parameter resolution in Iroha.

#### Objective

The objective of this proposal is to enable users to trace the origin and resolution of configuration parameters through logging.

#### Rationale

- **User Understanding:** By providing trace information, users gain a comprehensive understanding of how a configuration parameter's value is determined and whether it has been overridden from various sources.
- **Debugging Assistance:** Detailed trace logs help users diagnose and resolve configuration-related issues, as they can identify the specific sources contributing to the final parameter values.

#### Trace Events

The proposed trace mechanism aims to log two types of events:

1. **Override Event (DEBUG level):** This event records when a configuration parameter is overridden by another source. For example, if the value for the parameter "foo" was specified in the environment and also in the config file, a trace log will show:
   ```
   DEBUG: Override `foo` with env var `FOO`
   ```
2. **Resolution Event (TRACE level):** This event documents when a configuration parameter is resolved from a unique source. For example, if "bar" was provided only in the config file and "baz" was provided in another file that was resolved from "foo," the trace log will show:
   ```
   TRACE: `bar` set in `config.toml` TRACE: `baz` set in `~/.ssh/config` (source: `baz_source` set in `foo`)
   ```

#### Summary

By implementing the proposed trace mechanism, Iroha can offer users valuable trace information for configuration parameter resolution, improving transparency and simplifying debugging processes.

### 8. Introduce two-stage configuration resolution

This proposal aims to introduce two distinct configuration types, namely `UserConfig` and `ResolvedConfig`, for Iroha. The objective is to enhance configuration parsing, validation, and deprecation reporting while ensuring clear separation between user-provided configuration and the resolved, validated configuration data.

#### Objective

The objective of this proposal is to improve the configuration handling process by implementing two distinct configuration types, enabling efficient parsing and validation while facilitating flexible merging of multiple `UserConfig` instances. Additionally, the proposal aims to provide clear reporting of configuration deprecations during the resolution process.

#### Rationale

- **Parsing in "Parse-Not-Validate" Manner:** Designing the `ResolvedConfig` struct to be valid by definition, without requiring additional validation after construction, streamlines the configuration parsing process and reduces potential errors.
- **Minimalistic `ResolvedConfig`:** By including only necessary data in the `ResolvedConfig` struct, the configuration resolution becomes more efficient, as it contains precisely what is needed for the project's functionality.
- **Flexibility in `UserConfig` Merging:** Allowing optional fields in the `UserConfig` struct enables users to merge multiple configurations with ease, enhancing composability and adaptability for various scenarios.
- **Enhanced Deprecation Reporting:** The proposed design ensures that all configuration deprecations are reported during the resolution process, keeping users informed about changes and encouraging them to update their configurations accordingly.

### 9. Specify config path with a CLI argument

This proposal suggests adding a `--config` command-line argument to specify the configuration path for Iroha. Currently, the default value for the config path is `config.json`, which can be overridden using the `IROHA2_CONFIG_PATH` environment variable. However, to enhance user convenience and align with typical CLI app practices, this proposal advocates for including a CLI option to set the configuration path directly from the command line.

#### Objective

The objective of this proposal is to provide users with a convenient and intuitive way to specify the path to the configuration file when running the Iroha application via the command line. By introducing a `--config` CLI argument, users can easily customize the configuration path without relying solely on environment variables.

#### Rationale

- **User Convenience:** Providing a CLI option to set the configuration path enhances user convenience, allowing users to define the configuration path directly in the command-line invocation.
- **Standard CLI Practice:** Many command-line applications follow the convention of allowing users to specify configuration paths through CLI arguments, making it a familiar and expected feature.

## Implementation plan

The high-level plan for implementing the proposed points is the following:

### Step 1: Design Configuration Reference

This step includes the following points:

- Creating the reference before the implementation
- Using better aliases for config parameters
- Using consistent naming in configuration parameters
- Describing the configuration using TOML as a domain language
- Defining deprecation and migration policy as a part of the reference.

#### Dependencies/Prerequisites

- Collaboration with technical writers or domain experts to ensure accurate and comprehensive configuration reference documentation.
- A clear understanding of the configuration requirements and use cases to define a well-structured reference.

#### Potential Risks

- Miscommunication or lack of collaboration with technical writers may result in incomplete or inaccurate documentation.
- Inadequate understanding of configuration needs could lead to an ineffective reference.

### Step 2: Develop a Generic Configuration Rust Library

The objective of this step is to create a generic configuration Rust library that fulfills the requirements listed in the previous points:

- Support for exhaustive error messages, including batch-error collection and reliable deserialization failure messages.
- Ability to compose configurations from incomplete parts for easy merging and customization.
- Customizable merge strategies on a field-level to accommodate various configuration needs.
- Field-level tracing for setting and overriding, facilitating transparency in configuration resolution.
- Support for multiple configuration source formats, including but not limited to TOML.

_TODO: as this step is quite expanded and even has its own "objective", does it make sense to move it as a separate big point of the RFC?_

#### Dependencies/Prerequisites

- An advanced proficiency in Rust programming language, its ecosystem, and in proc macros in particular. These are crucial for developing the generic configuration library.
- A clear understanding of the specific requirements and functionalities outlined in the previous points to drive the library's design.

This step could be done in parallel with Step 1.

#### Potential Risks

- Developing a modular and flexible codebase that can accommodate all the specified features might be challenging, requiring careful design and implementation decisions.
- Balancing performance and flexibility could pose challenges, particularly when working with multiple configuration source formats.

#### Note

The development of a generic configuration Rust library allows for the potential reuse and adoption by other projects, making it a valuable contribution to the Rust ecosystem.

### Step 3: Re-implement `iroha_config` Crate

#### Dependencies/Prerequisites

- Completed configuration reference (Step 1) and a stable version of the developed configuration library (Step 2).
- Proficiency in Rust programming language to effectively re-implement the crate and utilize the generic configuration library.
- Knowledge of Test-Driven Development (TDD) methodologies to ensure thorough testing throughout the configuration implementation.
- Understanding of the Iroha domain to develop a reliable _resolution step_ that optimizes and strongly types the `ResolvedConfig`.

#### Potential Risks

- Introducing changes based on the configuration reference might cause compatibility issues with existing projects or require significant code adjustments.
- Properly applying TDD practices might might slow down development, requiring proper planning and time management.
- Designing an efficient and reliable resolution step could pose challenges due to the transformation of user-provided configurations into optimized and strongly-typed `ResolvedConfig`.
- Ensuring comprehensive test coverage for all configuration aspects might be time-consuming, especially when dealing with default values, environment variables, and various configuration scenarios.
- Having separate `UserConfig` and `ResolvedConfig` might lead to boilerplate code in the codebase, as both configurations need to be managed and validated separately.

### Step 4: Update the Rest Parts of the Codebase

The objective of this step is to seamlessly integrate the updated `iroha_config` crate into the core parts of Iroha that interact with configuration-related structures. This includes refactoring and adapting modules, components, and dependencies to leverage the enhanced features and stronger types provided by the updated configuration crate. Additionally, the main CLI application will be updated to support the `--config` command-line argument, offering users increased flexibility in configuring Iroha.

#### Dependencies/Prerequisites

- Successful completion of Step 3, ensuring the availability of the updated `iroha_config` crate with a reliable resolution step and, probably, comprehensive test coverage.
- Proficiency in Rust programming language and a solid understanding of Iroha's codebase to effectively update and refactor core components that interact with configuration-related structures.

#### Potential Risks

- Refactoring core components to integrate with the updated `iroha_config` crate may require careful planning and thorough testing to prevent any unforeseen issues or incompatibilities.
- Extending the main CLI application to support the `--config` argument needs precise implementation to ensure CLI parsing consistency.

#### Note

Updating the rest parts of the codebase is essential to achieve a cohesive and reliable configuration system throughout the entire Iroha project. While the addition of the `--config` CLI argument may seem minor, it offers users a convenient way to set the configuration path during runtime, enhancing user experience and flexibility. The updated `iroha_config` crate is designed to provide stronger types and improved structures, leading to better configuration management and more efficient code.

As with any significant update, it is crucial to conduct thorough testing and ensure compatibility to prevent any unintended regressions in the core components. However, this integration is expected to bring overall improvements to the configuration handling within the Iroha project.

## Impact assessment

### 1. Introduce Exhaustive Error Messages

**Impact:**

- Improved troubleshooting with detailed error messages.
- Transparent error reporting enhances user experience.
- User-friendly configuration with informative messages.

### 2. Use Better Aliases for Configuration Parameters

**Impact:**

- Enhanced clarity and readability of the configuration.
- Consistency and maintainability of configuration code.
- Simplified environment variable usage.

### 3. Use TOML

**Impact:**

- Simplified and human-friendly configuration reference.
- Enhanced tooling support and automation.
- Consistency in the Rust ecosystem.

### 4. Introduce Consistent Naming

**Impact:**

- Improved code readability and understanding.
- Consistency and easier maintenance.

### 5. Trace Configuration Resolution

**Impact:**

- Enhanced transparency and debugging.
- User confidence in configuration correctness.

### 6. Define Deprecation and Migration Policy for Configuration

**Impact:**

- Smooth configuration evolution and project sustainability.
- User adaptation to changes with clear communication and migration guidelines.

---

_TODO: describe impact of all other points_

## Conclusion

> - Summarize the key points discussed in the document.
> - Reinforce the benefits of implementing the proposed points.
> - Encourage further discussion and feedback.
