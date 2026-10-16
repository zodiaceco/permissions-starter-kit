# permissions-starter-kit

Out of the box starting point for managing [Zodiac Roles](https://docs.roles.gnosisguild.org) permissions

Keep the configuration of your Zodiac Roles Modifier as declarative statements in code.
The provided tooling automatically applies updates in a consistent and efficient way.

Learn more about the motivation behind this approach in our blog ["Permissions as Code"](https://engineering.gnosisguild.org/posts/permissions-as-code).

> [!TIP]
> **Need to manage more than role permissions?** Check out the [Zodiac Constellation Template](https://github.com/gnosisguild/zodiac-constellation-template) — a more powerful base for managing entire account constellations as code. Beyond updating role permissions, it lets you manage Safe owners, configure Delay mod setups, and deploy entire account setups in one go, rather than clicking through web interfaces.

> [!NOTE]
> **Already using this template?** GitHub template repositories don't auto-sync, so updates to this kit don't reach your project automatically. To pull in the latest changes (most recently, the switch to [`@zodiaceco/sdk`](https://www.npmjs.com/package/@zodiaceco/sdk)):
>
> ```bash
> git remote add upstream https://github.com/gnosisguild/permissions-starter-kit.git   # first time only
> git fetch upstream
> git checkout upstream/main -- .lib/scripts/sync-template.mjs
> node .lib/scripts/sync-template.mjs
> ```
>
> The script reconstructs the template version your repo was created from and merges from there, so only files where both you and the template changed the same lines will conflict. Your [roles/](./roles) directory is never overwritten — instead, the script migrates it (along with your contracts and ABIs) to the current tooling in a separate commit, and lists anything it couldn't migrate automatically.
>
> If the merge stops with conflicts, resolve them, `git commit`, and run the script once more to finish. Then run `yarn install` and `yarn setup`.
>
> This also applies if you synced before with an older version of the script or a plain `git merge`: run the commands above to finish the migration. `yarn apply` tells you when this is needed.

## Getting Started

[Use this template](https://github.com/new?template_name=permissions-starter-kit&template_owner=gnosisguild) to create a repository for your roles configuration.

### Initial Setup

After creating your own repository, clone it to work locally and open the project in a code editor such as [VSCode](https://code.visualstudio.com).

To initialize the project, open a terminal window and run the following command in the project root directory:  
 `yarn` _(If you encounter a 'command not found' error, ensure that you have [installed Yarn](https://classic.yarnpkg.com/lang/en/docs/install/) and try again.)_

### Creating a New Role

Roles are defined as folders in the [roles/](./roles) directory.
Folder names correspond to [role keys](#role-keys).
The template includes four example roles: `creditor_debtor`, `eth_wrapping`, `position_management`, and `swapper`.
You can either rename these folders or create new ones for your role.

Inside the role folder, create or edit the _permissions.ts_ file with the following boilerplate content:

```typescript
export default [
  // <- define your permissions here
] satisfies Permissions;
```

Also create or edit the _members.ts_ file, keeping the list of role member addresses:

```typescript
export default [
  // <- list all role member addresses here
] satisfies `0x${string}`[];
```

### Configuring Allowed Target Contracts

As a preparatory step before actually defining the role permissions, configure the addresses of all contracts you want to allow calling to.
Then, you can run a command that fetches the ABIs of these contracts.
This will provide automatic suggestions and correctness checks while authoring permissions.

Open [zodiac.config.ts](./zodiac.config.ts) in the editor and add labels and addresses of any contracts you plan to use as targets in your permissions.
At the top level of `contracts`, use the [chain prefix](#prefixed-addresses) of the host blockchain, e.g. `eth`, `gno`, `arb1`, or `base`.

Then, insert all target contract addresses as records using recognizable labels:

```typescript
import { defineConfig } from "@zodiaceco/sdk/cli/config";

export default defineConfig({
  contracts: {
    eth: {
      // <label>: "<contract address>",
      weth: "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2",

      // Optionally, group contracts in labeled categories:
      uniswap: {
        positions_nft: "0xC36442b4a4522E871399CD717aBDD847Ab11FE88",
      },
    },
  },
});
```

<a name="sdk-setup-steps"></a>

1. Save your changes in the zodiac.config.ts file and run the following command in the terminal window:  
   `yarn setup`  
   ABIs are stored in [abis/](./abis). If a contract isn't verified, the command prints the path where you should paste its ABI JSON by hand before running it again.
2. Open the VSCode command palette:  
   Mac: `Cmd` + `Shift` + `P`  
   Windows: `Ctrl` + `Shift` + `P`
3. Type `restart` and select the option `TypeScript: Restart TS server` from the suggestions list

### Edit Permissions

A role's _permissions.ts_ exports a list of **entries**. Each entry is one thing the role may do.

#### Contract Calls

In the _permissions.ts_ file for your role, type `allow.`.
You will see suggestions appearing next to your cursor.
Select the path to the target contract as previously defined in [zodiac.config.ts](./zodiac.config.ts).

<img src="https://i.imgur.com/2jKAoNk.gif" alt="Using suggestion to complete permissions" />

If the suggestions you see in the editor do not match the structure of your [zodiac.config.ts](./zodiac.config.ts) records, ensure you have followed the [three steps described above](#sdk-setup-steps).

#### Set Conditions on Parameters

To limit allowed values for individual function parameters, you can use the Roles Modifier's conditions system.
Condition functions are available under the global `c` variable.

Read more about conditions in the [documentation](https://docs.roles.gnosisguild.org/sdk/conditions).

#### Labelled Entries

Besides bare `allow` permissions, `@zodiaceco/sdk/actions` offers entries that carry a `label`, naming them as a single card in the Zodiac app (labels never reach the chain):

```typescript
import { custom, defikit, swap, transfer } from "@zodiaceco/sdk/actions";

export default [
  // A labelled bag of `allow` permissions, for anything the other entries don't cover.
  custom({
    label: "Wrap and unwrap ETH",
    permissions: [
      allow.eth.weth.deposit({ send: true }),
      allow.eth.weth.withdraw(),
    ],
  }),

  // A DeFi Kit preset, by protocol and verb.
  defikit.aave_v3.deposit({
    label: "Supply WETH to Aave v3",
    market: "Core",
    targets: ["WETH"],
  }),

  // Sign CoW orders selling any of `sell` for any of `buy`.
  swap({
    label: "Rebalance stables",
    sell: [USDC, DAI],
    buy: [USDC, DAI],
  }),

  // Send tokens to fixed recipients.
  transfer({
    label: "Payouts",
    tokens: [USDC],
    to: [PAYOUTS_SAFE],
  }),
] satisfies Permissions;
```

These entries describe what the role may do; they are compiled into permissions when the update is deployed.
That's why you don't call `defi-kit` yourself — a compiled permission set is not a valid entry.

### Edit Members

In the _members.ts_ file for your role, specify the list of addresses you want to assign the role to.

### Apply Updates

Once you have defined all members and permissions for the role, you can apply the update to your Roles mod.
If you need to set up a new Roles mod from scratch, refer to [this tutorial](https://www.zodiac.wiki/documentation/roles-modifier/roles-modifier-operator-tutorial).

In your terminal, run the following command:

```
yarn apply <role_key> <prefixed_address>
```

For example, to apply the role `eth_wrapping` to a mainnet Roles mod at address `0x1234123412341234123412341234123412341234`:

```
yarn apply eth_wrapping eth:0x1234123412341234123412341234123412341234
```

The first time you run it, the command authorizes this directory with your [Zodiac organization](https://app.zodiac.eco): it opens a browser to mint an API key and saves it to `.env`.

The command refreshes your organization data and contract ABIs, type-checks your roles, then pushes the update to your Zodiac workspace and prints a link to the Zodiac app, where you can review the changes and deploy them by signing the transaction.
The update links back to your repository (the web page of your `origin` git remote); set `ZODIAC_SOURCE` to name a different one.

Applying permissions for the first time will create a new role.
Subsequent applications will update the existing role, efficiently removing, updating, and adding permissions so that the role configuration on chain accurately reflects the permissions defined in code.

### Use New Role 🚀

Once applied, start using your role through [Zodiac Pilot](https://pilot.gnosisguild.org), our browser extension for streamlined Safe execution with support for roles and more complex account setups.

## Folder Structure and Conventions

- [zodiac.config.ts](./zodiac.config.ts) – Lists all contracts that are used as targets in permissions
- [abis/](./abis) – ABIs of these contracts, fetched by `yarn setup`
- [roles/](./roles) – Host directory for role configurations
  - [`role_key`/](./roles/eth_wrapping) – Each subfolder represents a distinct role. The folder name will be used as the [role key](#role-keys).
    - [members.ts](./roles/eth_wrapping/members.ts) – Assigns the role to the listed member addresses
    - [permissions.ts](./roles/eth_wrapping/permissions.ts) – Defines all permissions for this role

There are some additional files and folders in the template repository, which you won't usually need to edit.
They contain the necessary wiring for automatically applying the permissions.

##### Role Keys

In the Zodiac Roles Modifier, every role is identified by a `bytes32` string.
Choose a role key that accurately describes the purpose of the role.
We recommend using only the following characters for role keys: `a...z`, `0...9`, `_`
The length must be less than 32 characters.

##### Prefixed Addresses

The Roles tooling adopts [EIP-3770](https://eips.ethereum.org/EIPS/eip-3770) chain-specific addresses for identifying a contract on a specific chain in a compact way:

```
eth:0x1234123412341234123412341234123412341234
```

Chain prefixes for the supported chains are as follows:

- Mainnet: `eth`
- Gnosis Chain: `gno`
- Optimism: `oeth`
- Polygon: `matic`
- Polygon zkEVM: `zkevm`
- Arbitrum One: `arb1`
- Avalanche: `avax`
- Base: `base`
- BSC: `bnb`
- Celo: `celo`
- Sonic: `sonic`
- Berachain: `berachain`
- Unichain: `unichain`
- World Chain: `worldchain`
- BOB: `bob`
- Mantle: `mantle`
- Hemi: `hemi`
- Katana: `katana`
- Linea: `linea`
- Ink: `ink`
- HyperEVM: `hyperevm`
- Flare: `flare`
- Scroll: `scroll`
- Plasma: `plasma`
- MegaETH: `megaeth`
- Base Sepolia: `basesep`
- Sepolia Testnet: `sep`
