<template>
	<v-app id="inspire">
		<v-navigation-drawer
			v-model="drawer"
			app
			clipped
			>
			<v-list dense>
				<template v-for="source in sources">
					<v-list-item link :key="source">
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
			<v-dialog v-model="dialog" persistent max-width="500">
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
							</v-row>
						</v-container>
					</v-card-text>
					<v-card-actions>
						<v-spacer></v-spacer>
						<v-btn color="blue darken-1" text @click="dialog = false">Cancel</v-btn>
						<v-btn color="blue darken-1" text @click="doLogin()">Login</v-btn>
					</v-card-actions>
				</v-card>
			</v-dialog>
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
		dialog: null,

		login: null,
		password: null,

		jwt: null,
		sources: [],
	}),
	created () {
		this.$vuetify.theme.dark = true
	},
	methods: {
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
			this.dialog = false;

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
					await this.fetchSources();
				} else {
					alert("Login failed: " + JSON.stringify(resp_body.error))
				}
			}

		},
		fetchSources: async function() {
			const resp = await this.request("/sources", {}, "GET");
			const resp_data = await resp.json();
			console.log(resp_data);

			this.sources = resp_data.response;
		},
		focusPassword() {
			this.$refs["passwordInput"].focus();
		}
	}
}
</script>
