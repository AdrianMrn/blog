# Replacing Keytar with Electron's safeStorage in Ray

Written by [Adriaan Marain](https://twitter.com/AdriaanMrn)

[Ray](https://freek.dev/1868-introducing-ray-a-debugging-tool-for-pragmatic-developers) is an app we built at Spatie to make debugging your applications easier and faster. Being web developers, we naturally decided to write this app in Electron, which enabled us to move from nothing to a working prototype to a released product on 3 separate platforms within a matter of weeks.

About 9 months ago, [Alex](https://alexvanderbist.com/) added a [much requested feature](https://freek.dev/1921-debug-apps-running-on-remote-servers-using-ray) that allows you to connect to remote servers and receive their Ray outputs securely over SSH.

To save the credentials to a server, we needed to find a secure way to save the password or private key passphrase. We quickly settled on `node-keytar`, a native Node.js module that leverages your system's keychain (Keychain/libsecret/Credential Vault/…) to safely store passwords, hidden from other applications and users.

Using a native Node module brought one major disadvantage: we couldn't easily build for other platforms anymore without running the actual build on that platform. There are some solutions provided by `electron-builder`, Keytar, and other packages, but these all came with their own layer of overhead. We eventually decided to run the build on all platforms separately using the GitHub Actions CI.

Three weeks ago, on September 21, 2021, Electron 15 was released, and somewhat hidden in the release notes we found a mention to a newly added string encryption API: safeStorage ([PR](https://github.com/electron/electron/pull/30430)/[docs](https://www.electronjs.org/docs/latest/api/safe-storage)). Similarly to Keytar, Electron's safeStorage also uses the system's keychain to securely encrypt strings, but without the need for an extra dependency.

We jumped at the idea of simplifying our build process and removing a dependency, and wrote this simple implementation using safeStorage and `electron-store`, with an external API inspired by Keytar:

```typescript
import { safeStorage } from 'electron';
import Store from 'electron-store';

const store = new Store<Record<string, string>>({
  name: 'ray-encrypted',
  watch: true,
  encryptionKey: 'this_only_obfuscates',
});

export default {
  setPassword(key: string, password: string) {
    const buffer = safeStorage.encryptString(password);
    store.set(key, buffer.toString('latin1'));
  },

  deletePassword(key: string) {
    store.delete(key);
  },

  getCredentials(): Array<{ account: string; password: string }> {
    return Object.entries(store.store).reduce((credentials, [account, buffer]) => {
      return [...credentials, { account, password: safeStorage.decryptString(Buffer.from(buffer, 'latin1')) }];
    }, [] as Array<{ account: string; password: string }>);
  },
};

```

The only downside is that any previously saved passwords and passphrases saved using Keytar are now inaccessible for the app, but you can still find them by opening your system's keychain application and looking for any mentions of `ray`, `ssh_password_`, or `private_key_`. We also hope Ray wasn't the only spot you saved your server passwords.

Upon opening the servers overlay, existing users will receive a notification that their credentials need to be entered again. Ray will save which servers have not had their credentials updated yet, and will display this to the user. We used `electron-store`'s [migrations](https://github.com/sindresorhus/electron-store#migrations) feature for this:

```typescript
migrations: {
  '>=1.18.0': (store) => {
    store.set(
      'servers',
        store.get('servers').map((server) => ({ ...server, needsCredentialsUpdate: true }))
    );
  },
},
```
