# Hyperledger Fabric App using node.js SDK

By Victor Dods on behalf of LedgerDomain.

## Get Up And Running With A Single Command

In the project root, execute the following command in a terminal.

    make generated-artifacts && make initialization && make up

This brings up the blockchain network and web server; once you see

    www.skraukosaur.com | server listening at http://:::3000

you can then point your browser at `http://localhost:3000` to use the app's web client.

To bring the services down, hit Ctrl+C in the terminal.  The following command resets everything to
a completely clean state.

    make clean

Note that there are more minimal sets of commands that could be used for iteration during development, but
this one should suffice, if draconian.

## Prerequisites

It is apparently necessary to have Docker v1.13 (or presumably anything higher).  I was getting inexplicable (and
worse, nondeterministic) DNS resolution failures between services during GRPC calls with Docker v1.12.

This application uses Hyperledger Fabric v1.0.1.  The necessary components are:
-   hyperledger/fabric-peer:x86_64-1.0.1 docker image
-   hyperledger/fabric-orderer:x86_64-1.0.1 docker image
-   hyperledger/fabric-ca:x86_64-1.0.1 docker image
-   hyperledger/fabric-tools:x86_64-1.0.1 docker image
-   hyperledger/fabric-baseimage:x86_64-0.3.0 docker image

These will be retrieved automatically when bringing the various docker services up, but they can be pulled manually
ahead of time to avoid overhead in starting the application.

## Instructions To Run Application

Clone this repo with the following command.

    git clone https://github.com/vdods/FabricWebApp.git

As a one-time step, generate all necessary configuration and cryptographic material using the following command.

    make generated-artifacts

Among other things, this creates a docker volume `fabricwebapp_generated_artifacts__volume` containing all certs
and keys necessary for dispersal to the various components of this blockchain application.  In particular, this
creates a root CA (which for security purposes should remain offline because it has the root cert and key from
which all trust derives) for each organization which signs certs for the intermediate CAs (which will be online)
which create the certs/keys for the various peers, orderers, and users.

Then, as a one-time step, initialize the persistent volumes for each component of this application using the
following command.

    make initialization

This copies the necessary config/crypto materials to the volume for each component of the application (e.g.
CAs, orderer(s), peers, web server(s)).

Finally, as a repeatable step, bring up the application services using the following command.

    make up

The web app (in `web/server/app.js`) will automatically read the application and network configuration files
(`web/server/appcfg.json` and `web/server/netcfg.json` respectively), create necessary keystores, enroll all users
defined in `netcfg.json`, create all channels defined in `appcfg.json`, send participating organizations invitation
to join the respective channels, install/instantiate chaincode on the peers of each channel, and finally offer
a REST API and a simple web client on `http://localhost:3000`.  Point your web browser to this address to use
the web client.

Docker-compose is set to abort upon container exit, so if there's a problem, it will exit with a nonzero return
code, and the logs for the relevant services can be viewed.

## Web Client

The web client simply presents UI elements for invoking all the different available transactions on behalf of the
specified user ("Invoking User Name").  The event log is displayed on the right and shows the results of all
transactions.

The name of the privileged user is `Admin` -- this user can invoke any transaction, and only this user can invoke
`create_account` and `query_account_names`.  In order to invoke transactions as other users, the `create_account`
transaction must be successfully invoked by `Admin`.  For example,

    Invoking User Name: Admin
    Account Name:       Alice
    Initial Balance:    123
    Create Account (click the button)

After success, the user `Alice` can then be used as "Invoking User Name" to attempt to invoke transactions.
Certain transactions have certain constraints (e.g only `Admin` or the account owner can transfer an amount
out of an account).

## Structure Of Project

This application demonstrates three organizations (e.g. companies) participating together to provide a bank-type
application via simple web client where bank accounts can be created, balances queried, and transfers made between
accounts.  It's based on the `balance-transfer` example from [http://github.com/hyperledger/fabric-samples].

The components of this application (hostname and description) are:
-   Org0 components:
    -   <N/A> : Org0's (root) CA (certificate authority) which, to keep the root cert/key safe, is not run online
    -   ca.org0.skraukosaur.com : Org0's (intermediate) CA which is run online
    -   peer0.org0.skraukosaur.com : Org0's peer0 node
    -   peer1.org0.skraukosaur.com : Org0's peer1 node
-   Org1 components:
    -   <N/A> : Org1's (root) CA which, to keep the root cert/key safe, is not run online
    -   ca.org1.skraukosaur.com : Org1's (intermediate) CA which is run online
    -   peer0.org1.skraukosaur.com : Org0's peer0 node
    -   peer0.org1.skraukosaur.com : Org0's peer1 node
-   Org2 components:
    -   <N/A> : Org2's (root) CA which, to keep the root cert/key safe, is not run online
    -   <N/A> : Org2's (intermediate) CA which is not run online because it's not used in this app after initialization
    -   orderer.org2.skraukosaur.com : Org2's orderer node
-   Other components:
    -   www.skraukosaur.com : The web server (not formally associated with an organization like the others components are)

The `docker-compose.yaml` file and the supporting `docker/ca-base.yaml` and `docker/peer-base.yaml` files are what
define the services that run.

The source files are structured under the following directory structure:
-   `chaincode` : This is what is run on the peers to execute transactions
-   `docker` : Files defining docker-compose services
-   `source-artifacts` : Configuration files and script(s) used to create the generated artifacts described above
-   `web/server` : Node.js-based web server which drives the blockchain application and offers a simple REST API to the web client
-   `web/client` : Minimal web client to drive the blockchain app from the user side

Docker volumes:
-   `fabricwebapp_generated_artifacts__volume` : Generated by `make generated-artifacts`, and contains all CAs
    and CA-generated materials -- the `crypto-config` dir created by `fabric-ca-cryptogen.sh`, the `mychannel.tx`
    channel configuration used to create the channel for use by the blockchain app, and the `orderer.genesis.block`
    file used to initialize the blockchain with the participating organizations' credentials.
-   `fabricwebapp_com_skraukosaur_org0_ca__volume` : Org0's CA and state (user registration and enrollment)
-   `fabricwebapp_com_skraukosaur_org0_peer0__volume` : Org0's peer0 configuration and state
-   `fabricwebapp_com_skraukosaur_org0_peer1__volume` : Org0's peer1 configuration and state
-   `fabricwebapp_com_skraukosaur_org1_ca__volume` : Org1's CA and state (user registration and enrollment)
-   `fabricwebapp_com_skraukosaur_org1_peer0__volume` : Org1's peer0 configuration and state
-   `fabricwebapp_com_skraukosaur_org1_peer1__volume` : Org1's peer1 configuration and state
-   `fabricwebapp_com_skraukosaur_org2_ca__volume` : Org2's CA and state (user registration and enrollment), though this is
    not used by a running service
-   `fabricwebapp_com_skraukosaur_org2_orderer__volume` : Org2's orderer configuration and state
-   `fabricwebapp_com_skraukosaur_www__config_volume` : Crypto/config materials used by the www.skraukosaur.com web server
-   `fabricwebapp_com_skraukosaur_www__home_volume` : Keystore(s) and other persistent state used by the www.skraukosaur.com web server
-   `fabricwebapp_com_skraukosaur_www__node_modules_volume` : The `node_modules` directory used by the www.skraukosaur.com web server

## Finer-grained Instructions

There are several `make` targets that make controlling the docker-compose services defined in `docker-compose.yaml`
easier.

### Building and Doing Stuff

-   The command

        make generated-artifacts

    Creates the `fabricwebapp_generated_artifacts__volume` docker volume and generates all configuration and cryptographic
    materials necessary to later disperse (via `make initialization`) to the various components of the blockchain app.

-   The command

        make initialization

    Creates the `fabricwebapp_com_skraukosaur_*__volume` docker volumes and copies the necessary files from
    `fabricwebapp_generated_artifacts__volume` to the respective volumes.  This represents the communication of
    configuration/cryptographic material via out-of-bands channels to the participating servers.

-   The command

        make up

    Creates and starts all services, volumes, and networks and prints all services' logs to the console.  Hitting
    Ctrl+C in this mode will stop all services.  If any container exits, all will exit.

-   The command

        make up-detached

    does the same as `make up` except that it spins up all services in the background and does not print logs to the console.

-   The command

        make logs-follow

    can be used voluntarily, in the case that `make up-detached` was used, to follow the services' log printouts.
    Hitting Ctrl+C will detach from the log printout but not stop the services.  Note that currently the peers' state
    is not persisted between runs, so the blockchain will be lost upon stopping the containers.

-   The command

        make down

    brings down all services, but do not delete any persisted volumes (i.e. docker-based persistent storage).  Note that
    currently the peers' state is not persisted between runs, so the blockchain will be lost upon stopping the containers.

-   The command

        make down && make rm-state-volumes && make initialization && make up

    is a convenient single command you can use after stopping services to reset all services back to a clean state
    and restart them, following the services' logs.  There is typically no need to delete the persistent
    `fabricwebapp_com_skraukosaur_www__node_modules_volume` volume because it is not likely to qualitatively change or
    be corrupted (though that does happen sometimes during development for various reasons).  Note that the web
    server service in `docker-compose.yaml` does execute the command `npm install`, so any updates to
    `web/server/package.json` should automatically take effect.

-   The command

        make build-chaincode

    builds the chaincode in a separate service, and succeeds or fails quite quickly.  This is used to avoid
    the slow startup of the peers and other services needed to have the chaincode compile in the "real" way
    as in a production environment, and instead be able to iterate quickly when working on the chaincode.

### Viewing Stuff

-   The command

        make show-all-generated-resources

    will show all non-source resources that this project created that currently still exist.  See also
    the `make rm-all-generated-resources` command.

### Deleting Stuff

-   The command

        make rm-build-chaincode-state

    deletes the build artifacts generated by `make build-chaincode`, and can be used to force a "clean build"
    of the chaincode.

-   The command

        make rm-www-image

    deletes the docker image in which the www.skraukosaur.com service runs.

-   The command

        make rm-state-volumes

    deletes the persistent storage of the web server (in particular, the `fabricwebapp_webserver_tmp` and
    `fabricwebapp_webserver_homedir` volumes), and can be used for example to reset the web server to a 'clean'
    state, not having anything in the key/value store(s).  This can be executed only if the services are not up.
    WARNING: This will delete all of your webserver keystore data, and is IRREVERSIBLE!

-   The command

        make rm-node-modules

    deletes the node_modules directory of the web server (in particular, the `fabricwebapp_webserver_homedir_node_modules`
    volume).  This can be executed only if the services are not up.  This does not delete any real data that can't
    be easily recreated, though it may involve downloading a lot of node.js dependencies.

-   The command

        make rm-generated-artifacts

    will delete all generated artifacts.  In particular, it will delete the entire `generated-artifacts` directory.  This
    does not delete any real data that can't be easily recreated.

-   The command

        make rm-chaincode-docker-resources

    Deletes the docker containers created by the peers to run chaincode in net mode (which is default, as opposed to dev
    mode, specified by the `--peer-chaincodedev=true` option to the peer executable), as well as the docker images on which
    they run.  This shouldn't be necessary except to do a full clean-up, or when modifying chaincode (Fabric doesn't appear
    to detect that chaincode needs to be recompiled as of v1.0.0-alpha2).  This does not delete any real data that can't
    be easily recreated.

-   The command

        make rm-all-generated-resources

    or equivalently

        make clean

    will delete all non-source resources that this project created and currently still exist.  This should
    reset the project back to its original state with no leftover persistent data.  So be careful, because
    this will also delete your blockchain state and web server keystore.

## Testing

There is a simple shell script based test that can be run against the web server.  Bring the services up
(see above documentation on how), and then in a separate console, run `./test_rest_api.sh`.  It should
make several transactions and check their responses.  The script will succeed only if the responses were
as expected.  The test script must be run against a "fresh" server.

## Random Notes

-   The `.env` file contains the default environment variable values to be used in `docker-compose.yaml`.
    In particular:
    -   GENERAL_LOGGING_LEVEL controls how verbose the logging for each service is.
    -   TLS_ENABLED controls if TLS communication between each service is enabled.
-   __WARNING__: I have not been able to get the trustedRoots option for the TLS connection to the CA to work; thus
    I have set `"rejectUnauthorized":false` in `netcfg.json` to disable the check that the TLS server's cert is signed
    by a trusted CA.  The connection is still encrypted via TLS, but as is, it is vulnerable to man-in-the-middle attacks.

## To-Dos

-   Get the trustedRoots verification working for TLS connection to CA.  As noted in Random Notes, it is currently
    disabled, and therefore is vulnerable to man-in-the-middle attacks.
-   Use the fabric-ca-server-config.yaml files for ca.orgN.skraukosaur.com in fabric-ca-cryptogen.sh so that the CAs can
    be custom configured.
-   Verify that the state is preserved correctly between calls, and that the web server correctly handles existing
    persistent data in its keystore (regarding enrollment).
-   Create a single Makefile rule that ensures the correct artifacts/volumes/etc exist (creating if necessary),
    and then brings up all necessary services.  This rule should be able to be used to run the set of services again
    even if they exist already.  It should be the single, turn-key command that drives this entire example app.
-   Separate the `make generated-artifacts` step up into one command to generate the artifacts for each organization.
    This will require modifying `fabric-ca-cryptogen.sh` accordingly.

## Origin

This was originally a fork of https://github.com/ratnakar-asara/Fabric_SampleWebApp but has diverged quite a bit since then.
