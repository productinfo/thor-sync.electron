<template>
    <div class="pa-3">
        <div style="max-width: 800px; width: 100%; margin: 0 auto;">
            <v-subheader>General</v-subheader>
            <v-list two-line class="card-border" style="border-radius:2px;">
                <v-list-tile>
                    <v-list-tile-content>
                        <v-list-tile-title>Auto Update</v-list-tile-title>
                        <v-list-tile-sub-title>{{autoUpdateSubTitle}}</v-list-tile-sub-title>
                    </v-list-tile-content>
                    <v-list-tile-action>
                        <v-btn
                            style="text-transform:none"
                            flat
                            small
                            color="primary"
                            :disabled="!autoUpdateAction"
                            @click="autoUpdateAction"
                        >{{autoUpdateActionName}}</v-btn>
                    </v-list-tile-action>
                </v-list-tile>
                <v-divider/>
                <v-list-tile>
                    <v-list-tile-content>
                        <v-list-tile-title>Dark Theme</v-list-tile-title>
                    </v-list-tile-content>
                    <v-list-tile-action>
                        <v-switch
                            :disabled="darkThemeSwitchDisabled"
                            :inputValue="darkTheme"
                            @change="updateTheme($event)"
                            :ripple="false"
                        />
                    </v-list-tile-action>
                </v-list-tile>
                <v-divider/>
                <v-list-tile>
                    <v-list-tile-content>
                        <v-list-tile-title>Report an Issue</v-list-tile-title>
                        <v-list-tile-sub-title>To Github Issues Page</v-list-tile-sub-title>
                    </v-list-tile-content>
                    <v-list-tile-action>
                        <v-btn
                            style="text-transform:none"
                            flat
                            small
                            color="primary"
                            @click="openIssue"
                        >Open</v-btn>
                    </v-list-tile-action>
                </v-list-tile>
            </v-list>
            <v-layout align-end>
                <v-subheader>Nodes</v-subheader>
                <v-spacer/>
                <v-btn
                    class="caption"
                    small
                    flat
                    color="primary"
                    style="text-transform:none"
                    @click="addNode"
                >Add Node</v-btn>
            </v-layout>
            <v-list two-line class="card-border" style="border-radius:2px;">
                <template v-for="(node,i) in nodes">
                    <v-divider :key="i+'d'" v-if="i>0"/>
                    <v-list-tile :key="i">
                        <v-list-tile-content>
                            <v-list-tile-title>
                                <NetworkName class="mr-2" :genesis="node.genesis.id"/>
                                {{node.name}}
                            </v-list-tile-title>
                            <v-list-tile-sub-title>{{node.url}}</v-list-tile-sub-title>
                        </v-list-tile-content>
                        <v-list-tile-action>
                            <v-btn
                                v-if="!node.isPreset"
                                style="text-transform:none"
                                flat
                                small
                                color="primary"
                                @click="editNode(node)"
                            >Edit</v-btn>
                        </v-list-tile-action>
                    </v-list-tile>
                </template>
            </v-list>
        </div>
    </div>
</template>
<script lang="ts">
import { Vue, Component, Watch } from 'vue-property-decorator'
import { remote } from 'electron'
import NewNodeDialog from '../components/NewNodeDialog.vue'
import { presets } from '@/node-configs'

type Item = {
    title: string
    prepend?: {
        component: string
        props: {}
    }
    subTitle?: string
    actionName?: string
    action?: () => void
}

type Section = {
    title: string
    actionName?: string
    action?: () => void
    items: Item[]
}

const issueUrl = 'https://github.com/vechain/thor-sync.electron/issues'
const version = `${remote.app.getName()} v${remote.app.getVersion()} / Connex v${connex.version}`
const updateChecker = remote.app.EXTENSION.updateChecker

@Component
export default class Settings extends Vue {
    updater = {
        status: updateChecker.status,
        newVersion: updateChecker.newVersion,
        error: updateChecker.error
    }

    get autoUpdateStatusText() {
        if (this.updater.status === 'downloaded' && this.updater.newVersion) {
            return `New version ${this.updater.newVersion.version} available!`
        }
        if (this.updater.error) {
            return 'Error occurred'
        }
        if (this.updater.status === 'idle') {
            return 'Already up to date'
        }
    }

    get autoUpdateActionName() {
        switch (this.updater.status) {
            case 'idle': return 'Check'
            case 'checking': return 'Checking…'
            case 'downloading': return 'Downloading…'
            case 'downloaded': return 'Quit and Install'
        }
    }

    get autoUpdateAction() {
        switch (this.updater.status) {
            case 'idle': return () => {
                updateChecker.check()
                this.updater = {
                    status: updateChecker.status,
                    newVersion: updateChecker.newVersion,
                    error: updateChecker.error
                }
            }
            case 'checking': return undefined
            case 'downloading': return undefined
            case 'downloaded': return () => updateChecker.quitAndInstall()
        }
    }

    get autoUpdateSubTitle() {
        const statusText = this.autoUpdateStatusText
        if (statusText) {
            return `${version} (${statusText})`
        }
        return version
    }

    get nodes(): Array<NodeConfig & { isPreset: boolean }> {
        return presets.map(n => ({ ...n, isPreset: true }))
            .concat(this.$store.state.nodes.map((n: NodeConfig) => ({ ...n, isPreset: false })))
    }

    get darkTheme() {
        const result = (this.$store.state.preferences as entities.Preference[])
            .find(v => v.key === 'dark-theme')
        return result ? result.value as boolean : false
    }

    darkThemeSwitchDisabled = false

    updateTheme(dark: boolean) {
        this.darkThemeSwitchDisabled = true
        setTimeout(() => {
            this.darkThemeSwitchDisabled = false
        }, 1000)

        remote.app.EXTENSION.mainSettings.set('dark-theme', dark)
        GDB.transaction('rw', GDB.preferences, async () => {
            const result = await GDB.preferences.where({ key: 'dark-theme' }).limit(1).toArray()
            if (result.length > 0) {
                await GDB.preferences.where('key').equals('dark-theme').modify({ value: dark })
            } else {
                await GDB.preferences.put({ key: 'dark-theme', value: dark })
            }
        })
    }

    openIssue() {
        BUS.$emit('open-tab', {
            href: issueUrl
        })
    }
    addNode() {
        this.$dialog(NewNodeDialog, null)
    }

    editNode(node: NodeConfig) {
        this.$dialog(NewNodeDialog, node)
    }

    timer: any

    created() {
        this.timer = setInterval(() => {
            this.updater = {
                status: updateChecker.status,
                newVersion: updateChecker.newVersion,
                error: updateChecker.error
            }
        }, 2000)
    }
    destroyed() {
        clearInterval(this.timer)
    }
}
</script>

