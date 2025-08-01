<template>
	<div
		:style="`max-width: calc(100vw - ${
			$vuetify.display.lgAndUp ? 120 : 70
		}px)`"
		class="mx-2"
		v-if="node"
	>
		<v-row class="mt-2" align="center">
			<v-col style="min-width: 200px" class="ml-4">
				<span class="text-h6 text-grey">Device </span>
				<br />
				<span class="subtitle font-weight-bold font-monospace">
					{{ node.hexId }}
				</span>

				<v-icon
					@click="openLink(node.dbLink)"
					class="ml-2"
					v-tooltip:bottom="'See device config'"
				>
					open_in_new
				</v-icon>
				<br />
				<span
					v-if="$vuetify.display.smAndDown"
					class="comment font-weight-bold text-primary"
				>
					{{
						`${node.manufacturer || ''}${
							node.productDescription
								? ' - ' + node.productDescription
								: ''
						}`
					}}
				</span>
			</v-col>
			<v-col
				:class="$vuetify.display.smAndDown ? 'text-center' : 'text-end'"
			>
				<v-btn-group class="ml-2" multiple>
					<v-btn
						color="primary"
						variant="outlined"
						@click="toggleStatistics"
						:prepend-icon="statisticsOpeningIndicator"
						append-icon="multiline_chart"
					>
						Statistics
					</v-btn>
					<v-btn
						v-if="!node.isControllerNode"
						color="primary"
						@click.stop="pingNode(node)"
						variant="flat"
					>
						Ping
					</v-btn>
					<v-btn
						color="success"
						variant="flat"
						@click="advancedShowDialog = true"
					>
						Advanced
					</v-btn>
				</v-btn-group>
			</v-col>
		</v-row>

		<v-row v-if="nodeComments.length > 0">
			<v-col>
				<v-alert
					v-for="c in nodeComments"
					:key="c.level"
					text
					style="white-space: break-spaces"
					:type="c.level"
				>
					<span v-html="linkify(c.text)"></span>
				</v-alert>
			</v-col>
		</v-row>

		<v-row no-gutters>
			<v-sheet v-if="showStatistics" class="my-4" border rounded>
				<statistics-card title="Statistics" :node="node" />
			</v-sheet>
		</v-row>

		<v-divider class="my-4" />

		<div
			:class="[
				'd-flex',
				$vuetify.display.mdAndUp ? 'flex-row' : 'flex-column',
			]"
		>
			<v-tabs
				v-model="currentTab"
				show-arrows
				class="bg-transparent mb-4"
				:direction="
					$vuetify.display.mdAndUp ? 'vertical' : 'horizontal'
				"
			>
				<v-tab
					prepend-icon="widgets"
					value="node"
					class="justify-start"
					text="Node"
				/>
				<v-tab
					v-if="nodeMetadata"
					prepend-icon="help"
					value="manual"
					class="justify-start"
					text="Help"
					W
				/>
				<v-tab
					v-if="showHass"
					prepend-icon="home"
					value="homeassistant"
					class="justify-start"
					text="Home Assistant"
				/>
				<v-tab
					prepend-icon="device_hub"
					value="groups"
					class="justify-start"
					text="Groups"
				/>
				<v-tab
					v-if="node.schedule"
					prepend-icon="group"
					value="users"
					class="justify-start"
					text="Users"
				/>
				<v-tab
					value="ota"
					v-if="!node.isControllerNode"
					prepend-icon="auto_mode"
					class="justify-start"
					text="OTA Updates"
				/>
				<v-tab
					value="otw"
					v-if="node.isControllerNode"
					prepend-icon="auto_mode"
					class="justify-start"
					text="Firmware Updates"
				/>
				<v-tab
					prepend-icon="list_alt"
					value="events"
					class="justify-start"
					text="Events"
				/>
				<v-tab
					v-if="$vuetify.display.mdAndUp"
					prepend-icon="bug_report"
					class="justify-start"
					value="debug"
					text="Debug Info"
				/>
			</v-tabs>

			<!-- TABS -->
			<v-tabs-window
				style="background: transparent; padding-bottom: 10px"
				touchless
				class="fill"
				v-model="currentTab"
			>
				<!-- TAB NODE -->
				<v-tabs-window-item
					value="node"
					transition="slide-y-transition"
				>
					<node-details
						ref="nodeDetails"
						:headers="headers"
						:node="node"
						@updateValue="updateValue"
					></node-details>
				</v-tabs-window-item>

				<!-- TAB METADATA -->
				<v-tabs-window-item
					v-if="nodeMetadata"
					value="manual"
					transition="slide-y-transition"
				>
					<section
						v-for="meta in metaKeys"
						:key="`tab-${meta}`"
						class="px-8 py-4"
					>
						<h1 class="text-uppercase">{{ meta }}</h1>
						<p class="text-caption">
							<v-btn
								class="ma-2"
								v-if="meta === 'manual'"
								:href="nodeMetadata[meta]"
								variant="flat"
								color="primary"
							>
								DOWNLOAD
							</v-btn>
							<span v-else>
								{{ nodeMetadata[meta] }}
							</span>
						</p>
					</section>
				</v-tabs-window-item>

				<!-- TAB HOMEASSISTANT -->
				<v-tabs-window-item
					v-if="showHass"
					value="homeassistant"
					transition="slide-y-transition"
				>
					<home-assistant :node="node" :socket="socket" />
				</v-tabs-window-item>

				<!-- TAB GROUPS -->
				<v-tabs-window-item
					value="groups"
					transition="slide-y-transition"
				>
					<association-groups :node="node" />
				</v-tabs-window-item>

				<!-- TAB USERS -->
				<v-tabs-window-item
					v-if="node.schedule"
					value="users"
					transition="slide-y-transition"
				>
					<user-code-table :node="node" @updateValue="updateValue" />
				</v-tabs-window-item>

				<!-- TAB OTA UPDATES -->
				<v-tabs-window-item
					v-if="!node.isControllerNode"
					value="ota"
					transition="slide-y-transition"
				>
					<OTAUpdates :node="node" :socket="socket" />
				</v-tabs-window-item>

				<!-- TAB OTW UPDATES -->
				<v-tabs-window-item
					v-if="node.isControllerNode"
					value="otw"
					transition="slide-y-transition"
				>
					<OTWUpdates :node="node" :socket="socket" />
				</v-tabs-window-item>

				<!-- TAB EVENTS -->
				<v-tabs-window-item
					value="events"
					transition="slide-y-transition"
				>
					<v-container grid-list-md>
						<v-text-field
							v-model="searchEvents"
							variant="outlined"
							density="compact"
							prepend-icon="search"
							label="Search"
							class="pa-3"
							single-line
							hide-details
							style="max-width: 400px"
							clearable
						>
							<template #append>
								<v-btn
									v-if="!inverseSort"
									:variant="autoScroll ? 'flat' : 'outlined'"
									size="small"
									@click="toggleAutoScroll()"
									icon="autorenew"
									color="primary"
									v-tooltip:bottom="
										'Enable/Disable auto scroll'
									"
								/>

								<v-btn
									@click="toggleSort()"
									icon="swap_vert"
									class="ml-2"
									:variant="inverseSort ? 'flat' : 'outlined'"
									size="small"
									color="primary"
									v-tooltip:bottom="'Inverse Sort'"
								/>
							</template>
						</v-text-field>

						<v-col
							:ref="onEventsListInited"
							class="pa-5 events-list"
						>
							<div
								v-for="(event, index) in filteredNodeEvents"
								:key="'event_' + index + event.time"
								class="log-row font-monospace"
							>
								<span
									><i>{{
										getDateTimeString(event.time)
									}}</i></span
								>
								-
								<strong class="text-uppercase">{{
									event.event
								}}</strong>

								<span
									style="white-space: pre; font-size: 0.75rem"
									v-for="(arg, i) in event.args"
									:key="'arg_' + i"
									>{{ prettyPrintEventArg(arg, i) }}</span
								>
							</div>
						</v-col>
					</v-container>
				</v-tabs-window-item>

				<!-- TAB DEBUG INFO -->
				<v-tabs-window-item
					v-if="$vuetify.display.mdAndUp"
					value="debug"
					transition="slide-y-transition"
				>
					<v-container grid-list-md>
						<v-textarea
							class="debug-content font-monospace mx-2"
							append-icon="content_copy"
							v-model="nodeJson"
							readonly
							hide-details
							no-resize
							rows="37"
							ref="nodeJsonContent"
							@click:append="copyText"
						></v-textarea>
					</v-container>
				</v-tabs-window-item>
			</v-tabs-window>
		</div>

		<DialogAdvanced
			v-model="advancedShowDialog"
			@close="advancedShowDialog = false"
			:actions="advancedActions"
			@action="nodeAction"
		/>
	</div>
</template>

<script>
import { defineAsyncComponent } from 'vue'
import { jsonToList } from '@/lib/utils'
import { mapActions, mapState } from 'pinia'
import useBaseStore from '../../stores/base.js'
import { inboundEvents as socketActions } from '@server/lib/SocketEvents'
import InstancesMixin from '../../mixins/InstancesMixin.js'

import {
	SetValueStatus,
	setValueWasUnsupervisedOrSucceeded,
} from '@zwave-js/cc'
import { Protocols } from '@zwave-js/core'
import { nextTick } from 'vue'

export default {
	props: {
		actions: Array,
		headers: {
			type: Array,
			default: () => [],
		},
		node: Object,
		socket: Object,
	},
	mixins: [InstancesMixin],
	components: {
		AssociationGroups: defineAsyncComponent(
			() => import('@/components/nodes-table/AssociationGroups.vue'),
		),
		HomeAssistant: defineAsyncComponent(
			() => import('@/components/nodes-table/HomeAssistant.vue'),
		),
		NodeDetails: defineAsyncComponent(
			() => import('@/components/nodes-table/NodeDetails.vue'),
		),
		DialogAdvanced: defineAsyncComponent(
			() => import('@/components/dialogs/DialogAdvanced.vue'),
		),
		StatisticsCard: defineAsyncComponent(
			() => import('@/components/custom/StatisticsCard.vue'),
		),
		OTAUpdates: defineAsyncComponent(() => import('./OTAUpdates.vue')),
		OTWUpdates: defineAsyncComponent(() => import('./OTWUpdates.vue')),
		UserCodeTable: defineAsyncComponent(
			() => import('./UserCodeTable.vue'),
		),
	},
	computed: {
		...mapState(useBaseStore, ['gateway', 'mqtt']),
		isMobile() {
			return this.$vuetify.display.xs
		},
		isLongRange() {
			if (!this.node) return false

			return this.node.protocol === Protocols.ZWaveLongRange
		},
		nodeMetadata() {
			return this.node.deviceConfig?.metadata
		},
		nodeComments() {
			const comments = this.nodeMetadata?.comments ?? []

			return Array.isArray(comments) ? comments : [comments]
		},
		metaKeys() {
			const helpKeys = [
				'manual',
				'inclusion',
				'exclusion',
				'reset',
				'wakeup',
			]
			const keys = this.nodeMetadata ? Object.keys(this.nodeMetadata) : []

			return keys.filter((key) => helpKeys.includes(key))
		},
		nodeJson() {
			return JSON.stringify(this.node, null, 2)
		},
		showHass() {
			return (
				!this.mqtt.disabled &&
				this.gateway.hassDiscovery &&
				this.node.hassDevices &&
				Object.keys(this.node.hassDevices).length > 0
			)
		},
		statisticsOpeningIndicator() {
			return this.showStatistics ? 'arrow_drop_up' : 'arrow_drop_down'
		},
		statsBorderColor() {
			return this.showStatistics ? 'border-primary' : ''
		},
		filteredNodeEvents() {
			return this.node.eventsQueue
				.filter((event) => {
					return (
						!this.searchEvents ||
						JSON.stringify(event)
							.toLowerCase()
							.includes(this.searchEvents.toLowerCase())
					)
				})
				.sort((a, b) => {
					a = new Date(a.time)
					b = new Date(b.time)
					return this.inverseSort ? b - a : a - b
				})
		},
		advancedActions() {
			const nodeActions = this.node.isControllerNode
				? []
				: [
						{
							text: 'Firmware update',
							options: [
								{
									name: 'Begin',
									action: 'updateFirmware',
								},
								{
									name: 'Abort',
									action: 'abortFirmwareUpdate',
								},
							],
							icon: 'update',
							color: 'error',
							desc: 'Start/Stop a firmware update',
						},
						{
							text: 'Refresh Values',
							options: [
								{
									name: 'Refresh',
									action: 'refreshValues',
									args: {
										confirm:
											'Are you sure you want to refresh values of this node? This action increases network traffic',
									},
								},
							],
							icon: 'cached',
							desc: 'Update all CC values and metadata. Use only when many values seems stale',
						},
						{
							text: 'Re-interview Node',
							options: [
								{
									name: 'Interview',
									action: 'refreshInfo',
								},
							],
							icon: 'history',
							desc: 'Clear all info about this node and make a new full interview. Use when the node has wrong or missing capabilities',
						},
						{
							text: 'Failed Nodes',
							options: [
								{ name: 'Remove', action: 'removeFailedNode' },
								{
									name: 'Replace',
									action: 'replaceFailedNode',
								},
							],
							color: 'error',
							icon: 'dangerous',
							desc: 'Manage nodes that are dead and/or marked as failed with the controller',
						},
					]

			if (this.node.protocol !== Protocols.ZWaveLongRange) {
				nodeActions.splice(1, 0, {
					text: 'Rebuild Routes',
					options: [
						{
							name: 'Rebuild',
							action: 'rebuildNodeRoutes',
							args: {
								confirm:
									'Rebuilding node routes causes a lot of traffic, can take minutes up to hours and you can expect degraded performance while it is going on',
							},
						},
					],
					icon: 'healing',
					color: 'warning',
					desc: 'Discover and assign new routes from node to the controller and other nodes.',
				})
			}

			const nodeAssociation = this.node.isControllerNode
				? []
				: [
						{
							name: 'Clear',
							action: 'removeAllAssociations',
							color: 'error',
							args: {
								confirm:
									"This action will remove all associations of this node. This will also clear lifeline association with controller node, the node won't report state changes until that is set up again",
							},
						},
					]

			const CCActions = []

			if (this.node.supportsTime) {
				CCActions.push({
					text: 'Set Date and Time',
					options: [
						{
							name: 'Sync',
							action: 'syncNodeDateAndTime',
						},
					],
					icon: 'schedule',
					desc: 'Set date and time of this node to current time',
				})
			}

			return [
				{
					text: 'Export json',
					options: [
						{ name: 'UI', action: 'exportNode' },
						{ name: 'Driver', action: 'dumpNode' },
					],
					icon: 'get_app',
					desc: 'Export this node in a json file',
				},
				{
					text: 'Clear Retained',
					options: [
						{
							name: 'Clear',
							action: 'removeNodeRetained',
							args: {
								mqtt: true,
								confirm:
									'Are you sure you want to remove all retained messages?',
							},
						},
					],
					icon: 'clear',
					desc: 'All retained messages of this node will be removed from broker',
				},
				{
					text: 'Update topics',
					options: [
						{
							name: 'Update',
							action: 'updateNodeTopics',
							args: {
								mqtt: true,
								confirm:
									'Are you sure you want to update all topics?',
							},
						},
					],
					icon: 'update',
					desc: 'Update all node topics. Useful when name/location has changed',
				},
				...nodeActions,
				{
					text: 'Associations',
					options: [
						...nodeAssociation,
						{
							name: 'Remove',
							action: 'removeNodeFromAllAssociations',
							args: {
								confirm:
									'All direct associations to this node will be removed. Battery-powered nodes need to be woken up to edit their associations.',
							},
						},
					],
					icon: 'link_off',
					color: 'error',
					desc: 'Clear all node associations / Remove node from all associations',
				},
				...CCActions,
			]
		},
	},
	watch: {
		'node.eventsQueue': {
			handler() {
				this.scrollBottom()
			},
			deep: 1,
		},
		inverseSort() {
			this.savePreferences()
		},
		autoScroll() {
			this.savePreferences()
		},
	},
	data() {
		return {
			currentTab: 0,
			autoScroll: true,
			inverseSort: true,
			searchEvents: '',
			advancedShowDialog: false,
			showStatistics: false,
		}
	},
	mounted() {
		const pref = useBaseStore().getPreference('eventsList', {
			inverseSort: true,
			autoScroll: true,
		})

		this.inverseSort = pref.inverseSort
		this.autoScroll = pref.autoScroll
	},
	methods: {
		...mapActions(useBaseStore, [
			'showSnackbar',
			'setValue',
			'getDateTimeString',
		]),
		savePreferences() {
			useBaseStore().savePreferences({
				eventsList: {
					inverseSort: this.inverseSort,
					autoScroll: this.autoScroll,
				},
			})
		},
		async updateValue(v, customValue) {
			if (v) {
				// in this way I can check when the value receives an update
				v.toUpdate = true

				if (v.type === 'number') {
					v.newValue = Number(v.newValue)
				}

				// it's a button
				if (v.type === 'boolean' && !v.readable) {
					v.newValue = true
				}

				if (customValue !== undefined) {
					v.newValue = customValue
				}

				// update the value in store
				this.setValue(v)

				const response = await this.app.apiRequest('writeValue', [
					{
						nodeId: v.nodeId,
						commandClass: v.commandClass,
						endpoint: v.endpoint,
						property: v.property,
						propertyKey: v.propertyKey,
					},
					v.newValue,
					this.options,
				])

				v.toUpdate = false

				if (response.success) {
					const result = response.result
					const success = setValueWasUnsupervisedOrSucceeded(result)
					if (success) {
						this.showSnackbar('Value updated', 'success')
					} else {
						let reason = result.message
						if (
							!reason &&
							result.status === SetValueStatus.NoDeviceSupport
						) {
							reason = 'No device support'
						}
						this.showSnackbar(
							'Value update failed' +
								(reason ? ': ' + reason : ''),
							'error',
						)
					}
				} else {
					this.showSnackbar(
						`Error updating value${
							response.message ? ': ' + response.message : ''
						}`,
						'error',
					)
				}
			}
		},
		prettyPrintEventArg(arg, index) {
			return `\nArg ${index}:\n` + jsonToList(arg, undefined, 1)
		},
		linkify(text) {
			var urlRegex =
				/(\b(https?|ftp|file):\/\/[-A-Z0-9+&@#/%?=~_|!:,.;]*[-A-Z0-9+&@#/%=~_|])/gi
			return text.replace(urlRegex, function (url) {
				return '<a target="_blank" href="' + url + '">' + url + '</a>'
			})
		},
		copyText() {
			const textToCopy =
				this.$refs.nodeJsonContent.$el.querySelector('textarea')
			textToCopy.select()
			document.execCommand('copy')
		},
		nodeAction(action, args = {}) {
			if (action === 'exportNode') {
				this.exportNode()
			} else if (args.mqtt) {
				this.sendMqttAction(action, args.confirm)
			} else {
				this.sendAction(action, { ...args, nodeId: this.node.id })
			}
		},
		exportNode() {
			this.app.exportConfiguration(
				this.node,
				'node_' + this.node.id,
				'json',
			)
		},
		async sendMqttAction(action, confirmMessage) {
			if (this.node) {
				let ok = true

				if (confirmMessage) {
					ok = await this.app.confirm(
						'Info',
						confirmMessage,
						'info',
						{
							confirmText: 'Ok',
						},
					)
				}

				if (ok) {
					const args = [this.node.id]

					const data = {
						api: action,
						args: args,
					}
					this.socket.emit(socketActions.mqtt, data, (response) => {
						if (response.success) {
							this.showSnackbar(
								`Node ${this.node.id}: ${action} successfully sent `,
								'success',
							)
						} else {
							this.showSnackbar(
								`Error sending ${action} to node ${this.node.id}: ${response.message}`,
								'error',
							)
						}
					})
				}
			}
		},
		toggleStatistics() {
			this.showStatistics = !this.showStatistics
		},
		openLink(link) {
			window.open(link, '_blank')
		},
		toggleAutoScroll() {
			this.autoScroll = !this.autoScroll
		},
		toggleSort() {
			this.inverseSort = !this.inverseSort
		},
		onEventsListInited(element) {
			if (element) {
				this.eventsListEl = element.$el
				this.scrollBottom()
			} else {
				this.eventsListEl = null
			}
		},
		async scrollBottom() {
			if (!this.autoScroll || this.inverseSort) {
				return
			}
			const el = this.eventsListEl
			if (el) {
				await nextTick()
				el.scrollTop = el.scrollHeight
			}
		},
	},
}
</script>

<style>
.debug-content textarea {
	font-size: 0.75rem;
	line-height: 1.25 !important;
}
.font-monospace {
	font-family: 'Fira Code', monospace !important;
}

.events-list {
	max-height: 500px;
	height: 500px;
	overflow-y: scroll;
	border: 1px solid #ccc;
}

.log-row {
	cursor: default;
	padding: 0.5em 1em;
}

.log-row:nth-of-type(even) {
	background: v-bind('$vuetify.theme.current.colors.secondary');
	color: #000;
}

.log-row:hover {
	outline: 1px solid v-bind('$vuetify.theme.current.colors.secondary');
}
</style>
