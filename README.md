# TeamSpeak Declarations

This repository contains all kind of data which is related to TeamSpeak.

This data is used for code generation in various projects, therefore all the data is machine readable.

- [Errors.csv](Errors.csv): The error codes of TeamSpeak
- [Permissions.csv](Permissions.csv): The permissions in TeamSpeak
- [Messages.toml](Messages.toml): Commands, sent over TeamSpeak connections
- [Book.toml](Book.toml): Declarations for keeping track of all things which happen on a server
- [Enums.toml](Enums.toml): Various enums used in commands
- [MessagesToBook.toml](MessagesToBook.toml): Mappings from commands to book structs that allow to automatically update the tracked state
- [BookToMessages.toml](BookToMessages.toml): Functions that can be called on book structs and generate commands
- [Versions.csv](Versions.csv): A bunch of client versions where the versionHash is known
- [Badges.csv](Badges.csv): List of known badges

The low level TeamSpeak protocol is described in [ts3protocol.md](ts3protocol.md).

## License

Licensed under either of

 * [Apache License, Version 2.0](LICENSE-APACHE)
 * [MIT license](LICENSE-MIT)

at your option.
