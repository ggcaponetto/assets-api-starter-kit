# Assets API Starter Kit

This starter kit shows how you can start building a simple REST API to query assets data from [DB-sync](https://docs.cardano.org/cardano-components/cardano-db-sync/about-db-sync).

## Dev Environment

For executing this starter kit you'll need access to a running [DB-sync](https://docs.cardano.org/cardano-components/cardano-db-sync/about-db-sync) instance in sync with a Node running on the network of your preference.

In case you don't want to install the required components yourself, you can use [Demeter.run](https://demeter.run) platform to create a cloud environment with access to common Cardano infrastructure. The following command will open this repo in a private, web-based VSCode IDE with access to a running DB-Sync instance in the preview network.

[![Code in Cardano Workspace](https://demeter.run/code/badge.svg)](https://demeter.run/code?repository=https://github.com/txpipe/assets-api-starter-kit&template=typescript)

### Implementation Details

This Starter Kit implements a very simple REST API with a single GET endpoint for querying Assets information for a given `Policy Id`. 

The server is a [NodeJs](https://nodejs.org/en/) application using [ExpressJS](https://expressjs.com/) and Typescript. 

We have implemented the API following the clean architecture principles, making it easy to include new endpoints, data sources and use cases. 

For connecting with the database layer we have chosen to use [Prisma](https://www.prisma.io/) which provides us with a great development experience when it comes to access and querying db-sync. 

<img src="/assets/diagram.png" alt="diagram">

### Building & Running the Application

From the web-based workspace open a new Terminal and run the following commands for installing the dependencies:

```bash
$ npm install
```

Next we need to get Prisma initialized and connected against our db-sync instance. 
```bash
npx prisma init
```

Now we should have 2 new files generated in our repository:

* `prisma.schema`
* `.env`

If you open `prisma.schema` you will notice the schema is empty:
```typescript
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

For pulling the db-sync schema we are going to use the [Demeter.run](https://demeter.run) DB-Sync feature. 

Open your project Console in [Demeter.run](https://demeter.run) and go to the DB-Sync feature from the list.

<img src="assets/console-features.png" alt="db-sync features">

Go into the feature detail and copy the value of the `Connection string` field. 

<img src="assets/db-sync-settings.png" alt="db-sync settings">


You can now replace the content of the `DATABASE_URL` variable in the `.env` file generated by `npx prisma init` with this information:

```bash
# Environment variables declared in this file are automatically made available to Prisma.
# See the documentation for more detail: https://pris.ly/d/prisma-schema#accessing-environment-variables-from-the-schema

# Prisma supports the native connection string format for PostgreSQL, MySQL, SQLite, SQL Server, MongoDB and CockroachDB.
# See the documentation for all the connection string options: https://pris.ly/d/connection-strings

DATABASE_URL="postgresql://dmtrro:ZC0LwSZJnBZo60yOKoBIL0zjAIFt6OPTTiWRyZ6LZsWfZGn4Z2OGH6WW844GMVr8@dmtr-preview-postgres-repl.ftr-dbsync-v1.svc.cluster.local:5432/cardanodbsync?schema=public"
```

Once the `.env` file is updated go back to the terminal and run the following command:

```bash
npx prisma db pull
npx prisma generate
```

This command should have generated the high level Typescript objects for accessing DB-Sync from our code. 
If you check the `prisma.schema` file again you should see its now updated with the schema definition of db-sync. 

```typescript
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model ada_pots {
  id       BigInt  @id @default(autoincrement())
  slot_no  BigInt
  epoch_no Int
  treasury Decimal @db.Decimal(20, 0)
  reserves Decimal @db.Decimal(20, 0)
  rewards  Decimal @db.Decimal(20, 0)
  utxo     Decimal @db.Decimal(20, 0)
  deposits Decimal @db.Decimal(20, 0)
  fees     Decimal @db.Decimal(20, 0)
  block_id BigInt  @unique(map: "unique_ada_pots")
  block    block   @relation(fields: [block_id], references: [id], onDelete: Cascade, onUpdate: Restrict)
} ...
```

Feel free to explore the schema from this file, or you can use the `Schema` tab inside of the `DB-Sync` feature of [Demeter.run](https://demeter.run) for browsing the available tables. 

<img src="assets/db-sync-schema.png" alt="db-sync schema">

For our REST API we have implemented a data source in `assetsDataSource.ts`. You can check `DBSyncAssetsDataSource` for how we are using Prisma for querying DB-Sync and returning the Assets information mapped to a high-level object:

```typescript
async getForPolicyId(policyId: string): Promise<Asset[]> {
    const multiAssets = await this.client.multi_asset.findMany({
        where: {
            policy: Buffer.from(policyId, 'hex'),
        },
        include: {
            ma_tx_mint: true,
        }
    });
    return multiAssets.map((m) => {
        const asset = mapAsset(m);
        m.ma_tx_mint.forEach((t) => {
            asset.quantity += t.quantity.toNumber();
        });
        return asset;
    });
}
```

#### Build and Run

Once we have Prisma connected to our DB-Sync instance we can build and run the application. Go back to the terminal and execute the following commands:

```bash
npm run build
npm run dev
```

Your application should be now running in localhost:8000

### Testing the Application

If you want to test your API from outside of the workspace environment we need to expose the workspace port where the application is running.

For doing this in the [Demeter.run](https://demeter.run) console go to your workspaces, select the workspace where you are running the starter kit and go to the Exposed Ports tab. 

From this tab you can select to expose a new port. 

<img src="/assets/expose-port-new.png" alt="expose-port-new">

Once your port is exposed you can try a GET request to the auto-generated URL from your web browser

<img src="/assets/expose-port-list.png" alt="expose-port-list">

Example request with a policy id from the `preview` network:

```bash 
https://8000-slippery-sympathy-7n7uxh.us1.demeter.run/assets/policy/f5335f99c169cf2cbec9c6405750b533c011b24705590b800aa54cd6
```

The response we get for the given policy id:
```json
[{
	"id": "118",
	"fingerprint": "asset1uxw6angvk9tvaa79u98xw44tcqf6hflur3jr34",
	"name": "HOSKYt_Spectrum_Lq",
	"policyId": "f5335f99c169cf2cbec9c6405750b533c011b24705590b800aa54cd6",
	"quantity": 9223372036854776000
}]
```




