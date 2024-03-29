<template>
    <div id="sendalgo-app">
        <h3>Select wallet</h3>
        <div class="d-grid gap-2 mb-5">
            <button @click="connectToWalletConnect" class="btn btn-primary mr-3">WalletConnect</button>
            <button @click="connectToPeraWallet" class="btn btn-primary mr-3">Pera Wallet</button>
            <button @click="connectToDeflyWallet" class="btn btn-primary mr-3">Defly Wallet</button>
            <button @click="connectToSandNet" class="btn btn-primary mr-3">Sandbox (SandNet)</button>
        </div>
        <div v-if="this.sender !== ''" class="mb-5">
            <h3>Connected</h3>
            <p>
                Connection: <span>{{ this.connection }}</span>
            </p>
            <p>
                Network: <span>{{ this.network }}</span>
            </p>
            <p>
                Account: <span>{{ this.sender }}</span>
            </p>
            <button @click="disconnect" class="btn btn-primary">Disconnect</button>
        </div>
        <send-algo-form
            v-if="this.sender !== ''"
            :connection="this.connection"
            :walletclient="this.walletclient"
            :network="this.network"
            :sender="this.sender"
        />
    </div>
</template>

<script>
import WalletConnect from "@walletconnect/client";
import QRCodeModal from "algorand-walletconnect-qrcode-modal";
import { PeraWalletConnect } from "@perawallet/connect";
import { DeflyWalletConnect } from "@blockshake/defly-connect";
import kmd from "../kmd";

export default {
    data() {
        return {
            connection: "", // walletconnect | perawallet | defly
            walletclient: null,
            network: "", // network name
            sender: "", // connected account
        };
    },
    methods: {
        async connectToSandNet() {
            this.network = "SandNet";
            const accounts = await kmd.getSandboxAccounts();
            this.sender = accounts[0];
            this.connection = "sandbox";
        },
        async connectToWalletConnect() {
            // force connection to TestNet
            this.network = "TestNet";

            // Create a connector
            this.walletclient = new WalletConnect({
                bridge: "https://bridge.walletconnect.org", // Required
                qrcodeModal: QRCodeModal,
            });

            // // Kill existing session
            if (this.walletclient.connected) {
                await this.walletclient.killSession();
            }

            this.walletclient.createSession();

            // Subscribe to connection events
            this.walletclient.on("connect", (error, payload) => {
                if (error) {
                    throw error;
                }

                const { accounts } = payload.params[0];
                this.sender = accounts[0];
                this.connection = "walletconnect";
            });

            this.walletclient.on("session_update", (error, payload) => {
                if (error) {
                    throw error;
                }

                const { accounts } = payload.params[0];
                this.sender = accounts[0];
                this.connection = "walletconnect";
            });

            this.walletclient.on("disconnect", (error) => {
                if (error) {
                    throw error;
                }
            });
        },
        async connectToPeraWallet() {
            // force connection to TestNet
            this.network = "TestNet";

            // reconnect session if it exists
            try {
                this.walletclient = await new PeraWalletConnect();

                let accounts = await this.walletclient.reconnectSession();

                if (accounts.length <= 0) {
                    accounts = await this.walletclient.connect();
                }

                // disconnect listener
                this.walletclient.connector?.on("disconnect", (error) => {
                    if (error) {
                        throw error;
                    }
                });

                // you will need pera wallet instance to sign transactions
                this.sender = accounts[0];
                this.connection = "perawallet";
            } catch (error) {
                if (error?.data?.type !== "CONNECT_MODAL_CLOSED") {
                    // log the necessary errors
                }
            }
        },
        async connectToDeflyWallet() {
            // force connection to TestNet
            this.network = "TestNet";

            this.walletclient = new DeflyWalletConnect();

            // reconnect session if it exists
            try {
                let accounts = await this.walletclient.reconnectSession();

                if (accounts.length <= 0) {
                    accounts = await this.walletclient.connect();
                }

                this.walletclient.connector?.on("disconnect", (error) => {
                    if (error) {
                        throw error;
                    }
                });

                this.sender = accounts[0];
                this.connection = "deflywallet";
            } catch (error) {
                if (error?.data?.type !== "CONNECT_MODAL_CLOSED") {
                    // log the necessary errors
                }
            }
        },
        async disconnect() {
            switch (this.connection) {
                case "perawallet":
                case "deflywallet":
                    await this.walletclient.disconnect();
                    break;
                case "walletconnect":
                    await this.walletclient.killSession();
                    break;
                default:
                    break;
            }

            this.sender = "";
            this.connection = "";
            this.walletclient = null;
        },
    },
};
</script>
