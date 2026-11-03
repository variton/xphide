# xphide

A lightweight Python utility package for password hashing, symmetric encryption, JWT signing and verification, and environment variable access. It provides a small interface built on `bcrypt`, `cryptography`, and `python-jose`.

## Features

| Component | Purpose |
| --- | --- |
| `Crypt` | Hash passwords with bcrypt and verify password candidates. |
| `Xcrypt` | Encrypt and decrypt text with Fernet authenticated encryption. |
| `JwtCrypt` | Sign and verify JSON Web Tokens using HS256. |
| `load` | Read values from the process environment. |

## Installation

**Requires Python 3.14 or later.** Run the following command from the repository root:

```sh
python -m pip install -e .
```

For development, install the development dependency group with uv:

```sh
uv sync --group dev
```

## Usage

### Password hashing

Use `Crypt.hash` to create a salted bcrypt hash and `Crypt.is_equal` to verify a password against it. Pass the stored hash as the first argument when verifying:

```python
from xphide.crypt import Crypt

password_hash = Crypt.hash("example-password")

assert Crypt.is_equal(password_hash, "example-password")
assert not Crypt.is_equal(password_hash, "incorrect-password")
```

Both methods accept strings. Passwords must contain only ASCII characters, matching the encoding used by `Crypt`.

### Text encryption

Create an `Xcrypt` instance to encrypt and decrypt text. Encryption returns a `bytes` token; decryption returns a `str`.

```python
from xphide.xcrypt import Encoding, Xcrypt

crypt = Xcrypt(encoding=Encoding.UTF8)

token = crypt.encrypt("Café")
assert crypt.decrypt(token) == "Café"
```

The default encoding is ASCII. Select `Encoding.UTF8` for text containing non-ASCII characters. The runtime return type of `encrypt` is `bytes`, despite its current `str` annotation.

#### Reusing an encryption key

Each default `Xcrypt` instance generates an independent Fernet key. To decrypt data across instances or application restarts, retain the key securely and provide a cipher initialized with that key:

```python
from cryptography.fernet import Fernet
from xphide.xcrypt import Xcrypt

# Generate once, then store securely and retrieve for subsequent use.
key = Fernet.generate_key()

writer = Xcrypt(cipher=Fernet(key))
reader = Xcrypt(cipher=Fernet(key))

assert reader.decrypt(writer.encrypt("secret")) == "secret"
```

Invalid or tampered tokens, including tokens encrypted with a different key, raise `cryptography.fernet.InvalidToken`.

### JWT signing and verification

`JwtCrypt` signs and verifies tokens using HS256. Set the `SJWTK` environment variable **before importing `xphide.jwtcrypt`** to configure the default signing key:

```sh
export SJWTK='replace-with-your-private-signing-key'
```

```python
from xphide.jwtcrypt import JwtCrypt

token = JwtCrypt.encode({"sub": "user-123"})
payload = JwtCrypt.decode(token)

assert payload["sub"] == "user-123"
```

The default key is read at module import time. Both methods also accept an explicit, nonempty `_key` argument that overrides the default:

```python
import os

from xphide.jwtcrypt import JwtCrypt

signing_key = os.environ["APP_SIGNING_KEY"]
token = JwtCrypt.encode({"sub": "user-123"}, _key=signing_key)
payload = JwtCrypt.decode(token, _key=signing_key)
```

JWT payloads are signed, not encrypted. Invalid tokens or signatures raise `jose.exceptions.JWTError`; expired tokens raise its subclass, `jose.exceptions.ExpiredSignatureError`. Keep signing keys private and out of version control.

### Environment variables

Use `load` to read a named environment variable, or omit the argument to read `SJWTK`:

```python
from xphide.envvar_loader import load

signing_key = load()  # Reads SJWTK.
custom_value = load("MY_VARIABLE")
```

The function returns `None` when a variable is absent and an empty string when it is defined but empty. It reads the process environment and does not load `.env` files.

## Development

The source code is located in `src/xphide/`, with tests in `tests/`. After installing the development dependencies, run the test suite from the repository root:

```sh
export SJWTK='local-test-signing-key'
uv run pytest
```

The JWT tests require a nonempty `SJWTK` supplied by the calling environment before the test process starts. They fail if it is missing or empty.

## License

Released under the [MIT License](LICENSE).
