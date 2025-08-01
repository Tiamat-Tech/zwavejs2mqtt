<template>
	<v-container fluid class="pa-4">
		<v-data-table
			:headers="headers"
			:items="items"
			class="elevation-1"
			style="margin-bottom: 80px"
			:search="search"
			v-model:options="tableOptions"
			:sort-by="tableOptions.sortBy"
		>
			<template #top>
				<v-col class="pt-0">
					<h2 class="pa-3">Provisioning Entries</h2>
					<missing-keys-alert />
					<v-menu
						v-model="showInfoTooltip"
						location="bottom left"
						:style="{
							position: 'fixed',
							left: x + 'px',
							top: y + 'px',
						}"
					>
						<v-list density="compact">
							<v-list-item density="compact">
								<v-list-item-title>
									When an entry has a Node associated it
									cannot be edited
								</v-list-item-title>
							</v-list-item>
						</v-list>
					</v-menu>

					<v-row>
						<v-col cols="12" sm="6">
							<v-text-field
								v-model="search"
								clearable
								flat
								variant="outlined"
								hide-details
								single-line
								class="ma-2"
								style="max-width: 300px; min-width: 250px"
								prepend-inner-icon="search"
								label="Search"
								append-icon="refresh"
								@click:append="refreshItems"
							></v-text-field>
						</v-col>
					</v-row>
				</v-col>
			</template>

			<template #[`item.nodeId`]="{ item }">
				<v-btn
					v-if="item.nodeId"
					color="primary"
					size="small"
					rounded
					@click.stop="showNodeDialog(item)"
				>
					{{ item.nodeId }}

					<v-icon class="ml-2" size="small">open_in_new</v-icon>
				</v-btn>
			</template>

			<template #[`item.dsk`]="{ item }">
				<span v-if="streamerMode">*********</span>
				<span v-else> {{ item.dsk }} </span>
			</template>

			<template #[`item.status`]="{ item }">
				<v-switch
					v-model="item.status"
					@update:model-value="onChange(item)"
					density="compact"
					hide-details
				></v-switch>
			</template>

			<template #[`item.protocol`]="{ item }">
				<v-btn
					v-if="
						item.supportedProtocols?.includes(
							Protocols.ZWaveLongRange,
						)
					"
					@click="toggleProtocol(item)"
					:disabled="!!item.nodeId"
					density="compact"
					rounded
					@focus="onRowFocus($event, item)"
					@blur="onRowBlur($event, item)"
					@mouseenter="onRowFocus($event, item)"
					@mouseleave="onRowBlur($event, item)"
					size="small"
					variant="outlined"
					:color="getProtocolColor(item)"
					>{{ getProtocol(item) }}</v-btn
				>
				<span class="text-caption" v-else> Z-Wave </span>
			</template>

			<template #[`item.securityClasses.s2AccessControl`]="{ item }">
				<div
					@focus="onRowFocus($event, item)"
					@blur="onRowBlur($event, item)"
					@mouseenter="onRowFocus($event, item)"
					@mouseleave="onRowBlur($event, item)"
				>
					<v-checkbox
						v-model="item.securityClasses.s2AccessControl"
						@update:model-value="onChange(item)"
						:disabled="
							!!item.nodeId ||
							!item.requestedSecurityClasses.s2AccessControl
						"
						hide-details
						density="compact"
					></v-checkbox>
				</div>
			</template>
			<template #[`item.securityClasses.s2Authenticated`]="{ item }">
				<div
					@focus="onRowFocus($event, item)"
					@blur="onRowBlur($event, item)"
					@mouseenter="onRowFocus($event, item)"
					@mouseleave="onRowBlur($event, item)"
				>
					<v-checkbox
						v-model="item.securityClasses.s2Authenticated"
						@update:model-value="onChange(item)"
						:disabled="
							!!item.nodeId ||
							!item.requestedSecurityClasses.s2Authenticated
						"
						hide-details
						density="compact"
					></v-checkbox>
				</div>
			</template>
			<template #[`item.securityClasses.s2Unauthenticated`]="{ item }">
				<div
					@focus="onRowFocus($event, item)"
					@blur="onRowBlur($event, item)"
					@mouseenter="onRowFocus($event, item)"
					@mouseleave="onRowBlur($event, item)"
				>
					<v-checkbox
						v-if="item.protocol === Protocols.ZWave"
						v-model="item.securityClasses.s2Unauthenticated"
						:disabled="
							!!item.nodeId ||
							!item.requestedSecurityClasses.s2Unauthenticated
						"
						@update:model-value="onChange(item)"
						hide-details
						density="compact"
					></v-checkbox>
					<span v-else></span>
				</div>
			</template>
			<template #[`item.securityClasses.s0Legacy`]="{ item }">
				<div
					@focus="onRowFocus($event, item)"
					@blur="onRowBlur($event, item)"
					@mouseenter="onRowFocus($event, item)"
					@mouseleave="onRowBlur($event, item)"
				>
					<v-checkbox
						v-if="item.protocol === Protocols.ZWave"
						v-model="item.securityClasses.s0Legacy"
						:disabled="
							!!item.nodeId ||
							!item.requestedSecurityClasses.s0Legacy
						"
						@update:model-value="onChange(item)"
						hide-details
						density="compact"
					></v-checkbox>
					<span v-else></span>
				</div>
			</template>
			<template #[`item.actions`]="{ item }">
				<v-icon
					class="mr-2"
					size="small"
					color="error"
					@click="removeItem(item)"
					>delete</v-icon
				>
				<v-icon size="small" color="success" @click="editItem(item)"
					>edit</v-icon
				>
			</template>
		</v-data-table>
		<base-fab
			v-model="fab"
			location="bottom end"
			transition="slide-y-reverse-transition"
			class="mb-7"
			icon-open="menu"
			icon-close="close"
			:items="fabItems"
		/>

		<v-dialog
			:fullscreen="$vuetify.display.xs"
			max-width="1200px"
			v-model="expandedNodeDialog"
			persistent
			@keydown.exit="closeDialog()"
		>
			<v-card min-height="90vh">
				<v-fab
					style="position: absolute; right: 5px; top: 5px"
					size="x-small"
					@click="closeDialog()"
					icon="close"
				/>
				<v-card-text class="pt-3">
					<expanded-node :node="expandedNode" :socket="socket" />
				</v-card-text>
			</v-card>
		</v-dialog>
	</v-container>
</template>
<script>
import { defineAsyncComponent } from 'vue'
import { tryParseDSKFromQRCodeString, Protocols } from '@zwave-js/core'
import { mapActions } from 'pinia'
import {
	parseSecurityClasses,
	validDsk,
	securityClassesToArray,
	getProtocol,
	getProtocolColor,
} from '../lib/utils.js'
import useBaseStore from '../stores/base.js'
import InstancesMixin from '../mixins/InstancesMixin.js'
import { protocolsItems } from '../lib/items.js'
import { socketEvents } from '@server/lib/SocketEvents'

export default {
	name: 'SmartStart',
	props: {
		socket: Object,
	},
	mixins: [InstancesMixin],
	components: {
		BaseFab: defineAsyncComponent(
			() => import('@/components/custom/BaseFab.vue'),
		),
		ExpandedNode: defineAsyncComponent(
			() => import('@/components/nodes-table/ExpandedNode.vue'),
		),
		MissingKeysAlert: defineAsyncComponent(
			() => import('@/components/custom/MissingKeysAlert.vue'),
		),
	},
	watch: {
		tableOptions: {
			handler() {
				useBaseStore().savePreferences({
					smartStartTable: this.tableOptions,
				})
			},
			deep: true,
		},
	},
	computed: {
		streamerMode() {
			return useBaseStore().ui.streamerMode
		},
		fabItems() {
			return [
				{
					icon: 'add',
					color: 'primary',
					action: () => this.editItem(),
					tooltip: 'Add',
				},
				{
					icon: 'qr_code_scanner',
					color: 'warning',
					action: () => this.scanItem(),
					tooltip: 'Scan',
				},
				{
					icon: 'refresh',
					color: 'success',
					action: () => this.refreshItems(),
					tooltip: 'Refresh',
				},
				{
					icon: 'file_download',
					color: 'error',
					action: () => this.importList(),
					tooltip: 'Import',
				},
				{
					icon: 'file_upload',
					color: 'purple',
					action: () => this.exportList(),
					tooltip: 'Export',
				},
			]
		},
	},
	data() {
		return {
			items: [],
			showInfoTooltip: false,
			x: 0,
			y: 0,
			Protocols,
			fab: false,
			search: '',
			tableOptions: {
				sortBy: [{ key: 'nodeId' }],
			},
			expandedNode: null,
			expandedNodeDialog: false,
			headers: [
				{ title: 'ID', key: 'nodeId' },
				{ title: 'Name', key: 'name' },
				{ title: 'Location', key: 'location' },
				{ title: 'Active', key: 'status' },
				{ title: 'Protocol', key: 'protocol' },
				{ title: 'DSK', key: 'dsk' },
				{
					title: 'S2 Access Control',
					key: 'securityClasses.s2AccessControl',
				},
				{
					title: 'S2 Authenticated',
					key: 'securityClasses.s2Authenticated',
				},
				{
					title: 'S2 Unauthenticated',
					key: 'securityClasses.s2Unauthenticated',
				},
				{ title: 'S0 Legacy', key: 'securityClasses.s0Legacy' },
				{ title: 'Manufacturer', key: 'manufacturer' },
				{ title: 'Label', key: 'label' },
				{ title: 'Description', key: 'description' },
				{ title: 'Actions', key: 'actions', sortable: false },
			],
			edited: false,
		}
	},
	mounted() {
		this.tableOptions = useBaseStore().getPreference('smartStartTable', {
			page: 1,
			itemsPerPage: 10,
			sortBy: [{ key: 'nodeId' }],
			groupBy: [],
		})

		this.refreshItems()

		this.bindEvent(socketEvents.nodeAdded, () => {
			this.refreshItems()
		})
	},
	beforeUnmount() {
		if (this.socket) {
			this.unbindEvents()
		}
	},
	methods: {
		...mapActions(useBaseStore, ['showSnackbar']),
		onRowFocus(event, item) {
			if (item.nodeId) {
				// get mouse position
				this.x = event.clientX + 10
				this.y = event.clientY + 10

				this.showInfoTooltip = true
			}
		},
		onRowBlur() {
			this.showInfoTooltip = false
		},
		getProtocol,
		getProtocolColor(node) {
			return getProtocolColor(node, this.app.currentTheme)
		},
		showNodeDialog(entity) {
			const node = this.nodes.find((n) => n.id === entity.nodeId)
			if (node) {
				this.expandedNode = node
				this.expandedNodeDialog = true
			}
		},
		closeDialog() {
			this.expandedNode = null
			this.expandedNodeDialog = false
		},
		async refreshItems() {
			const response = await this.app.apiRequest(
				'getProvisioningEntries',
				[],
				{
					infoSnack: false,
					errorSnack: true,
				},
			)

			if (response.success) {
				this.items = this.parseItems(response.result)
			}
		},
		async exportList() {
			await this.app.exportConfiguration(
				this.items,
				'provisioningEntries',
				'json',
			)
		},
		async importList() {
			const { data } = await this.app.importFile('json')

			if (data) {
				for (const entry of data) {
					this.updateItem(entry)
				}
			}
		},
		async onChange(item) {
			await this.updateItem(item)
			this.edited = false
			this.refreshItems()
		},
		async toggleProtocol(item) {
			item.protocol =
				item.protocol === Protocols.ZWave
					? Protocols.ZWaveLongRange
					: Protocols.ZWave
			await this.onChange(item)
		},
		async updateItem(itemOrQr) {
			const response = await this.app.apiRequest(
				'provisionSmartStartNode',
				[
					typeof itemOrQr === 'string'
						? itemOrQr
						: this.convertItem(itemOrQr),
				],
			)

			if (response.success) {
				const entry = response.result
				this.showSnackbar(
					`Node ${entry.nodeId || entry.name || ''} ${
						this.edited ? 'updated' : 'added'
					}`,
					'success',
				)

				this.refreshItems()
			}
		},
		async scanItem() {
			let qrString = await this.app.confirm(
				'New entry',
				'Scan QR Code or import it as an image',
				'info',
				{
					qrScan: true,
					canceltext: 'Close',
					width: 500,
				},
			)

			if (qrString) {
				const dsk = tryParseDSKFromQRCodeString(qrString)
				if (dsk) {
					this.showSnackbar(
						`Provided QR Code is not a SmartStart QR Code.\nIn order to use it go to Control Panel > Manage Nodes and use Scan QR Code Inclusion.`,
						'warning',
					)
				} else {
					this.updateItem(qrString)
				}
			}
		},
		async editItem(existingItem) {
			if (existingItem) {
				this.edited = true
			}

			let inputs = [
				{
					type: 'text',
					label: 'Name',
					required: true,
					key: 'name',
					hint: 'The node name',
					default: existingItem ? existingItem.name : '',
				},
				{
					type: 'text',
					label: 'Location',
					required: true,
					key: 'location',
					hint: 'The node location',
					default: existingItem ? existingItem.location : '',
				},
			]

			if (!existingItem?.nodeId) {
				inputs = [
					...inputs,
					{
						type: 'text',
						label: 'DSK',
						required: true,
						key: 'dsk',
						hint: 'Enter the full DSK code (dashes included) for your device',
						rules: [validDsk],
						default: existingItem ? existingItem.dsk : '',
					},
					{
						type: 'list',
						label: 'Protocol',
						required: true,
						key: 'protocol',
						items: protocolsItems,
						disabled:
							existingItem &&
							(!existingItem.supportedProtocols ||
								existingItem.supportedProtocols.length < 2),
						hint: 'Inclusion protocol to use',
						default: existingItem
							? existingItem.protocol
							: Protocols.ZWave,
					},
					{
						type: 'checkbox',
						label: 'S2 Access Control',
						key: 's2AccessControl',
						disabled: existingItem
							? !existingItem.requestedSecurityClasses
									.s2AccessControl
							: false,
						default: existingItem
							? existingItem.securityClasses.s2AccessControl
							: false,
					},
					{
						type: 'checkbox',
						label: 'S2 Authenticated',
						key: 's2Authenticated',
						disabled: existingItem
							? !existingItem.requestedSecurityClasses
									.s2Authenticated
							: false,
						default: existingItem
							? existingItem.securityClasses.s2Authenticated
							: false,
					},
					{
						type: 'checkbox',
						label: 'S2 Unauthenticated',
						key: 's2Unauthenticated',
						disabled: existingItem
							? !existingItem.requestedSecurityClasses
									.s2Unauthenticated
							: false,
						default: existingItem
							? existingItem.securityClasses.s2Unauthenticated
							: false,
						show: (item) => {
							return item.protocol === Protocols.ZWave
						},
					},
					{
						type: 'checkbox',
						label: 'S0 Legacy',
						key: 's0Legacy',
						disabled: existingItem
							? !existingItem.requestedSecurityClasses.s0Legacy
							: false,
						default: existingItem
							? existingItem.securityClasses.s0Legacy
							: false,
						show: (item) => {
							return item.protocol === Protocols.ZWave
						},
					},
				]
			}

			let item = await this.app.confirm(
				(existingItem ? 'Update' : 'New') + ' entry',
				'',
				'info',
				{
					confirmText: existingItem ? 'Update' : 'Add',
					width: 500,
					inputs,
				},
			)

			if (item.dsk || existingItem?.nodeId) {
				if (!existingItem?.nodeId) {
					const securityClasses = {
						s2Unauthenticated: item.s2Unauthenticated,
						s2Authenticated: item.s2Authenticated,
						s2AccessControl: item.s2AccessControl,
						s0Legacy: item.s0Legacy,
					}
					delete item.s2AccessControl
					delete item.s2Authenticated
					delete item.s2Unauthenticated
					delete item.s0Legacy

					item.securityClasses = securityClasses
				}

				if (existingItem) {
					// extend existing item props that are not shown in dialog
					item = { ...existingItem, ...item }
				}

				this.updateItem(item)
			} else if (Object.keys(item).length > 0 && !item.dsk) {
				this.showSnackbar('DSK is required', 'error')
			}
		},
		async removeItem(item) {
			if (
				await this.app.confirm(
					'Attention',
					`Are you sure you want to delete this item from provisioning? Removing it from provisioning will not exclude the node`,
					'alert',
				)
			) {
				const response = await this.app.apiRequest(
					'unprovisionSmartStartNode',
					[item.dsk],
				)

				if (response.success) {
					this.showSnackbar(`Node ${item.nodeId} removed`, 'success')
					this.refreshItems()
				}
			}
		},
		parseItems(items) {
			return items.map((item) => {
				return {
					...item,
					status: !item.status,
					protocol: item.protocol ?? Protocols.ZWave,
					securityClasses: parseSecurityClasses(
						item.securityClasses,
						false,
					),
					requestedSecurityClasses: parseSecurityClasses(
						item.requestedSecurityClasses,
						item.requestedSecurityClasses ? false : true,
					),
				}
			})
		},
		convertItem(item) {
			item = {
				...item,
				status: item.status ? 0 : 1,
				protocol: item.protocol ?? Protocols.ZWave,
				securityClasses: securityClassesToArray(item.securityClasses),
				requestedSecurityClasses: securityClassesToArray(
					item.requestedSecurityClasses ||
						parseSecurityClasses([], true),
				),
			}

			return item
		},
	},
}
</script>
