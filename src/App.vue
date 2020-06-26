<template>
	<v-app id="inspire">
		<v-navigation-drawer
			v-model="drawer"
			app
			clipped
			>
			<v-list dense>
				<template v-for="source in sourceNames">
					<v-list-item link :key="source" @click="setCurrentSource(source)">
						<v-list-item-action>
							<v-icon>mdi-folder-move</v-icon>
						</v-list-item-action>
						<v-list-item-content>
							<v-list-item-title>{{ source }}</v-list-item-title>
						</v-list-item-content>
					</v-list-item>
				</template>
			</v-list>
		</v-navigation-drawer>

		<v-app-bar
			app
			clipped-left
			>
			<v-app-bar-nav-icon @click.stop="drawer = !drawer"></v-app-bar-nav-icon>
			<v-toolbar-title>File-Sorter</v-toolbar-title>

			<v-spacer></v-spacer>
			<v-dialog v-model="loginDialog" persistent max-width="500" v-if="jwt == null">
				<template v-slot:activator="{on, attrs}">
					<v-btn color="primary" dark v-bind="attrs" v-on="on">
						Login
					</v-btn>
				</template>
				<v-card>
					<v-card-title>
						<span class="headline">Login</span>
					</v-card-title>
					<v-card-text>
						<v-container>
							<v-row>
								<v-col cols="12">
									<v-text-field label="Login" v-model="login" required v-on:keyup.enter="focusPassword()"></v-text-field>
								</v-col>
								<v-col cols="12">
									<v-text-field label="Password" required v-model="password" type="password" v-on:keyup.enter="doLogin()" ref="passwordInput"></v-text-field>
								</v-col>
								<v-switch v-model="savePassword" class="ma-2" label="Keep me logged in"></v-switch>
							</v-row>
						</v-container>
					</v-card-text>
					<v-card-actions>
						<v-spacer></v-spacer>
						<v-btn color="blue darken-1" text @click="loginDialog = false">Cancel</v-btn>
						<v-btn color="blue darken-1" text @click="doLogin()">Login</v-btn>
					</v-card-actions>
				</v-card>
			</v-dialog>
			<v-btn color="primary" dark v-if="jwt" @click="logout()">
				Logout
			</v-btn>
		</v-app-bar>

		<v-main>
			<v-container
				class="fill-height"
				fluid
				>
				<v-row
					align="center"
					justify="center"
					>
					<v-list v-if="currentFiles != null && currentFiles.length > 0" min-width="30%">
						<v-list-item v-for="file in currentFiles" :key="file" link>
							<v-menu offset-y>
								<template v-slot:activator="{ on, attrs }">
									<v-list-item-icon>
										<v-icon>mdi-file</v-icon>
									</v-list-item-icon>
				<v-list-item-content>
					<v-list-item-title v-text="file" v-on="on" v-bind="attrs"></v-list-item-title>
				</v-list-item-content>
								</template>
								<v-list>
									<v-list-item
										v-for="destination in destinations"
										:key="destination"
										@click="moveFile(file, destination)"
										>
										<v-list-item-title>{{ destination }}</v-list-item-title>
									</v-list-item>
								</v-list>
							</v-menu>
						</v-list-item>
					</v-list>
				</v-row>
			</v-container>
		</v-main>

		<v-footer app>
			<span>&copy; 2020</span>
		</v-footer>
	</v-app>
</template>

<script>
const ENDPOINT = "http://localhost:8000";

export default {
	props: {
		source: String,
	},
	data: () => ({
		drawer: null,
		loginDialog: null,

		login: null,
		password: null,

		jwt: null,
		savePassword: true,
		sources: {},
		currentSource: null,

		destinations: [],
	}),
	computed: {
		sourceNames: function() {
			const names = Object.getOwnPropertyNames(this.sources).filter((value) => value != "__ob__").sort();
			return names;
		},
		currentFiles: function() {
			if(this.currentSource) {
				return this.sources[this.currentSource];
			} else {
				return null;
			}
		}
	},
	watch: {
		sourceNames: function(sourceNames) {
			if(sourceNames.length > 0) {
				this.currentSource = sourceNames[0];
			}
		}
	},
	created () {
		this.$vuetify.theme.dark = true
	},
	async mounted() {
		if(localStorage.jwt) {
			this.jwt = localStorage.jwt;
			await this.fetchSources();
			await this.fetchDests();
		}
	},
	methods: {
		setCurrentSource: function(source) {
			this.currentSource = source;
		},
		request: async function(route, data, method) {
			if(method === "GET") {
				return await fetch(ENDPOINT + route, {
					method: "GET",
					headers: {'Content-Type': 'application/json', 'authorization': 'Bearer ' + this.jwt},
				})
			} else {
				return await fetch(ENDPOINT + route, {
					method: method,
					headers: {'Content-Type': 'application/json', 'authorization': 'Bearer ' + this.jwt},
					body: JSON.stringify(data),
				})
			}
		},
		doLogin: async function() {
			this.loginDialog = false;

			const resp = await fetch(ENDPOINT + "/login", {
				method: "POST", 
				headers: {'Content-Type': 'application/json'},
				body: JSON.stringify({password: this.password, username: this.login})
			});
			if(!resp.ok) {
				alert("Login request failed: ", resp.statusText);
			} else {
				const resp_body = await resp.json();
				if(resp_body.status === "Success") {
					this.jwt = resp_body.response.token;
					if(this.savePassword) {
						localStorage.jwt = this.jwt;
					}
					await this.fetchSources();
					await this.fetchDests();
				} else {
					alert("Login failed: " + JSON.stringify(resp_body.error))
				}
			}

		},
		logout() {
			this.jwt = null;
			this.sources = {};
		},
		fetchSources: async function() {
			const resp = await this.request("/files", {}, "GET");
			const resp_data = await resp.json();

			this.sources = resp_data.response.files;
		},
		fetchDests: async function() {
			const resp = await this.request("/destinations", {}, "GET");
			const resp_data = await resp.json();

			this.destinations = resp_data.response;
		},
		moveFile: async function(name, destination){
			await this.request("/files/" + this.currentSource + "/" + name + "/" + destination, {}, "PUT");
			
			await this.fetchSources();
		},
		focusPassword() {
			this.$refs["passwordInput"].focus();
		}
	}
}
</script>
