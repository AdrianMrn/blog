# Replacing node-keytar with Electron's safeStorage in Ray

[Ray](https://freek.dev/1868-introducing-ray-a-debugging-tool-for-pragmatic-developers) is an app we built at Spatie to make debugging your applications easier and faster. Being web developers, we naturally decided to write this app in Electron, which enabled us move from nothing to a working prototype to a released product on 3 separate platforms within a matter of weeks.

About 9 months ago, [Alex](https://alexvanderbist.com/) added a [much requested feature](https://freek.dev/1921-debug-apps-running-on-remote-servers-using-ray) which allows you to connect to remote servers and receive their Ray outputs securely over SSH.

To save the credentials to a server, we needed to find a secure way to save the password or private key passphrase. We quickly settled on node-keytar, a native Node.js module that leverages your system's keychain (Keychain/libsecret/Credential Vault/…) to safely store passwords, hidden from other applications and users.

Using a native Node module brought one major disadvantage: we couldn't easily build for other platforms anymore, without running the actual build on that platform. There are some solutions provided by electron-builder, node-keytar, and other packages, but these all came with their own layer of overhead.

Three weeks ago, onSeptember 21, 2021, Electron 15 was released, and somewhat hidden in the release notes we found a mention to a newly added string encryption API: safeStorage ([PR](https://github.com/electron/electron/pull/30430)/[docs](https://www.electronjs.org/docs/latest/api/safe-storage)). Similarly to node-keytar, Electron's safeStorage also uses the system's keychain to securely encrypt strings, but without the need for an extra dependency.

We jumped at the idea of simplifying our build process and removing a dependency, and wrote this simple implementation using safeStorage and electron-store, with an external API inspired by node-keytar:

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

The only downside is that any previously saved passwords and passphrases saved using node-keytar are now inaccessible for the app, but you can still get them by opening your system's keychain application and looking for any mentions of "ray", "ssh_password_", or "private_key_".
