<template>
    <div
        :class="{
            'kiwi-wrap--statebrowser-drawopen': stateBrowserDrawOpen,
            'kiwi-wrap--monospace': setting('useMonospace'),
            'kiwi-wrap--touch': state.ui.is_touch,
        }"
        :data-activebuffer="buffer ? buffer.name.toLowerCase() : ''"
        class="kiwi-wrap kiwi-theme-bg"
        @click="emitDocumentClick"
        @paste="emitBufferPaste"
    >
        <link :href="themeUrl" rel="stylesheet" type="text/css">

        <template v-if="!hasStarted || (!fallbackComponent && networks.length === 0)">
            <component :is="startupComponent" @start="startUp"/>
        </template>
        <template v-else>
            <state-browser :networks="networks" :ui-state="uiState"/>
            <div class="kiwi-workspace" @click="stateBrowserDrawOpen = false">
                <div class="kiwi-workspace-background"/>

                <template v-if="!activeComponent && network">
                    <container
                        :network="network"
                        :buffer="buffer"
                        :users="users"
                        :ui-state="uiState"
                    >
                        <media-viewer
                            v-if="mediaviewerOpen"
                            slot="before"
                            :url="mediaviewerUrl"
                            :component="mediaviewerComponent"
                            :is-iframe="mediaviewerIframe"
                        />
                    </container>
                    <control-input :container="networks" :buffer="buffer"/>
                </template>
                <component
                    v-else-if="!activeComponent"
                    :is="fallbackComponent"
                    v-bind="fallbackComponentProps"
                />
                <component v-else :is="activeComponent" v-bind="activeComponentProps"/>
            </div>
        </template>
    </div>
</template>

<script>

import 'font-awesome-webpack';
import '@/res/globalStyle.css';
import Tinycon from 'tinycon';
import Vue from 'vue';

import startupWelcome from '@/components/startups/Welcome';
import startupZncLogin from '@/components/startups/ZncLogin';
import startupCustomServer from '@/components/startups/CustomServer';
import startupKiwiBnc from '@/components/startups/KiwiBnc';
import startupPersonal from '@/components/startups/Personal';
import StateBrowser from '@/components/StateBrowser';
import AppSettings from '@/components/AppSettings';
import Container from '@/components/Container';
import ControlInput from '@/components/ControlInput';
import MediaViewer from '@/components/MediaViewer';
import * as Notifications from '@/libs/Notifications';
import * as AudioBleep from '@/libs/AudioBleep';
import * as bufferTools from '@/libs/bufferTools';
import ThemeManager from '@/libs/ThemeManager';
import Logger from '@/libs/Logger';
import state from '@/libs/state';
import InputHandler from '@/libs/InputHandler';

let log = Logger.namespace('App.vue');

/* eslint-disable no-new */
new InputHandler(state);

// ContainerUiState gets passed around to child components so they all know
// what state the UI is in. Ie. sidebar open or closed, what section of the
// sidebar is open, etc.
let ContainerUiState = Vue.extend({
    data() {
        return {
            sidebarOpen: false,
            sidebarPinned: false,
            // sidebarSection may be either '', 'user', 'settings', 'nicklist'
            sidebarSection: '',
            sidebarUser: null,
        };
    },
    computed: {
        isPinned() {
            // Pinned sidebar only works on full width windows otherwise its too small to see
            return this.sidebarPinned && this.canPin;
        },
        isOpen() {
            return !this.isPinned && this.sidebarOpen;
        },
        isClosed() {
            return !this.isOpen && !this.isPinned;
        },
        canPin() {
            return state.ui.app_width > 769;
        },
    },
    methods: {
        section() {
            if (this.isClosed) {
                return '';
            }

            let section = this.sidebarSection;
            let isChannel = state.getActiveBuffer().isChannel();

            if (section === 'settings' && isChannel) {
                return 'settings';
            } else if (section === 'user' && this.sidebarUser && isChannel) {
                return 'user';
            } else if (section === 'nicklist' && isChannel) {
                return 'nicklist';
            }

            return '';
        },
        pin() {
            this.sidebarPinned = true;
            if (this.sidebarSection === '') {
                this.sidebarSection = 'nicklist';
            }
        },
        unpin() {
            this.sidebarPinned = false;
            this.close();
        },
        close() {
            this.sidebarOpen = false;
            this.sidebarSection = '';
            this.sidebarUser = null;
        },
        showUser(user) {
            this.sidebarUser = user;
            this.sidebarOpen = true;
            this.sidebarSection = 'user';
        },
        showNicklist() {
            this.sidebarOpen = true;
            this.sidebarSection = 'nicklist';
        },
        showBufferSettings() {
            this.sidebarOpen = true;
            this.sidebarSection = 'settings';
        },
    },
});

export default {
    components: {
        StateBrowser,
        Container,
        ControlInput,
        MediaViewer,
    },
    data: function data() {
        return {
            startupComponent: null,
            hasStarted: false,
            // When on mobile screens, the statebrowser turns into a drawer
            stateBrowserDrawOpen: false,
            // If set, will become the main view instead of a buffer/nicklist container
            activeComponent: null,
            activeComponentProps: {},
            // If set, will become the main view when no networks are available to be shown
            // and there is no active component set
            fallbackComponent: null,
            fallbackComponentProps: {},
            mediaviewerOpen: false,
            mediaviewerUrl: '',
            mediaviewerComponent: null,
            mediaviewerIframe: false,
            themeUrl: '',
            uiState: new ContainerUiState(),
        };
    },
    computed: {
        state() {
            return state;
        },
        networks() {
            return state.networks;
        },
        network() {
            return state.getActiveNetwork();
        },
        buffer() {
            return state.getActiveBuffer();
        },
        users() {
            let activeNetwork = this.network;
            if (!activeNetwork) {
                return null;
            }

            return activeNetwork.users;
        },
    },
    created: function created() {
        this.listen(state, 'active.component', (component, props) => {
            this.activeComponent = null;
            if (component) {
                this.activeComponentProps = props;
                this.activeComponent = component;
            }
        });
        this.listen(state, 'statebrowser.toggle', () => {
            this.stateBrowserDrawOpen = !this.stateBrowserDrawOpen;
        });
        this.listen(state, 'statebrowser.show', () => {
            this.stateBrowserDrawOpen = true;
        });
        this.listen(state, 'statebrowser.hide', () => {
            this.stateBrowserDrawOpen = false;
        });
        this.listen(state, 'mediaviewer.show', (url) => {
            let opts = {};

            // The passed url may be a string or an options object
            if (typeof url === 'string') {
                opts = { url: url };
            } else {
                opts = url;
            }

            this.mediaviewerUrl = opts.url;
            this.mediaviewerComponent = opts.component;
            this.mediaviewerIframe = opts.iframe;
            this.mediaviewerOpen = true;
        });
        this.listen(state, 'mediaviewer.hide', () => {
            this.mediaviewerOpen = false;
        });

        let themes = ThemeManager.instance();
        this.themeUrl = ThemeManager.themeUrl(themes.currentTheme());
        this.listen(state, 'theme.change', () => {
            this.themeUrl = ThemeManager.themeUrl(themes.currentTheme());
        });

        document.addEventListener('keydown', event => this.onKeyDown(event), false);
        window.addEventListener('focus', (event) => {
            state.ui.app_has_focus = true;
            let buffer = state.getActiveBuffer();
            if (buffer) {
                buffer.markAsRead(true);
            }

            state.ui.favicon_counter = 0;
        }, false);
        window.addEventListener('blur', (event) => {
            state.ui.app_has_focus = false;
        }, false);
        window.addEventListener('touchstart', (event) => {
            // Parts of the UI adjust themselves if we're known to be using a touchscreen
            state.ui.is_touch = true;
        });

        // Track the window dimensions into the reactive ui state
        function trackWindowDims() {
            state.ui.app_width = window.innerWidth;
            state.ui.app_height = window.innerHeight;
        }
        window.addEventListener('resize', trackWindowDims);
        trackWindowDims();

        // favicon bubble
        Tinycon.setOptions({
            width: 7,
            height: 9,
            color: '#ffffff',
            background: '#b32d2d',
            fallback: true,
        });
        state.$watch('ui.favicon_counter', (newVal) => {
            if (newVal) {
                Tinycon.setBubble(newVal);
            } else {
                Tinycon.reset();
            }
        });
        this.listen(state, 'message.new', (message) => {
            if (!message.isHighlight || state.ui.app_has_focus) {
                return;
            }

            state.ui.favicon_counter++;
        });
        if (this.uiState.canPin && state.setting('sidebarPinned')) {
            this.uiState.pin();
        }
    },
    mounted: function mounted() {
        // Decide which startup screen to use depending on the config
        let startupScreens = {
            welcome: startupWelcome,
            customServer: startupCustomServer,
            kiwiBnc: startupKiwiBnc,
            znc: startupZncLogin,
            personal: startupPersonal,
        };
        let extraStartupScreens = state.getStartups();

        let startupName = state.settings.startupScreen || 'personal';
        let startup = extraStartupScreens[startupName] || startupScreens[startupName];

        if (!startup) {
            Logger.error(`Startup screen "${startupName}" does not exist`);
        } else {
            this.startupComponent = startup;
        }
    },
    methods: {
        // Triggered by a startup screen event
        startUp(opts) {
            log('startUp()');
            if (opts && opts.fallbackComponent) {
                this.fallbackComponent = opts.fallbackComponent;
            }
            if (opts && opts.fallbackComponentProps) {
                this.fallbackComponentProps = opts.fallbackComponentProps;
            }

            // Make sure a startup screen can't trigger these more than once
            if (!this.hasStarted) {
                this.warnOnPageClose();
                Notifications.requestPermission();
                Notifications.listenForNewMessages(state);
                AudioBleep.listenForHighlights(state);
            }

            this.hasStarted = true;
        },
        warnOnPageClose() {
            window.onbeforeunload = () => {
                if (state.setting('warnOnExit')) {
                    return this.$t('window_unload');
                }

                return null;
            };
        },
        emitBufferPaste(event) {
            // bail if no buffer is active, or the buffer is hidden by another component
            if (!this.state.getActiveBuffer() || this.activeComponent !== null) {
                return;
            }

            // bail if a sidebar is open
            if (this.$data.uiState.sidebarOpen) {
                return;
            }

            state.$emit('buffer.paste', event);
        },
        emitDocumentClick(event) {
            state.$emit('document.clicked', event);
        },
        onKeyDown(event) {
            state.$emit('document.keydown', event);

            let meta = false;

            if (navigator.appVersion.indexOf('Mac') !== -1) {
                meta = event.metaKey;
            } else {
                meta = event.ctrlKey;
            }

            if (meta && event.keyCode === 221) {
                // meta + ]
                let buffer = bufferTools.getNextBuffer();
                if (buffer) {
                    state.setActiveBuffer(buffer.networkid, buffer.name);
                }
                event.preventDefault();
            } else if (meta && event.keyCode === 219) {
                // meta + [
                let buffer = bufferTools.getPreviousBuffer();
                if (buffer) {
                    state.setActiveBuffer(buffer.networkid, buffer.name);
                }
                event.preventDefault();
            } else if (meta && event.keyCode === 79) {
                // meta + o
                state.$emit('active.component', AppSettings);
                event.preventDefault();
            } else if (meta && event.keyCode === 83) {
                // meta + s
                let network = state.getActiveNetwork();
                if (network) {
                    network.showServerBuffer('settings');
                }
                event.preventDefault();
            }
        },
        setting(name) {
            return state.setting(name);
        },
    },
};
</script>

<style lang="less">
html {
    height: 100%;
    margin: 0;
    padding: 0;
}

body {
    height: 100%;
    margin: 0;
    padding: 0;
}

.kiwi-wrap {
    font-size: 90%;
    line-height: 1.6em;
    font-family: Source Sans Pro, Helvetica, sans-serif;
    -webkit-font-smoothing: antialiased;
    height: 100%;
    overflow: hidden;
}

.kiwi-wrap--monospace {
    font-family: Consolas, monaco, monospace;
    font-size: 80%;
}

.kiwi-workspace {
    position: relative;
    margin-left: 220px;
    left: 0;
    display: block;
    height: 100%;
    transition: left 0.2s, margin-left 0.2s;
}

.kiwi-workspace::before {
    position: absolute;
    content: '';
    right: 0;
    left: 0;
    top: 0;
    height: 7px;
    z-index: 0;
}

/* When the statebrowser opens as a draw, darken the workspace */
.kiwi-workspace::after {
    position: fixed;
    top: 0;
    right: 0;
    content: '';
    overflow: hidden;
    opacity: 0;
    transition: opacity 0.5s;
    will-change: opacity;
}

.kiwi-workspace-background {
    position: absolute;
    top: 0;
    left: 0;
    height: 100%;
    width: 100%;
    z-index: -1;
}

.kiwi-statebrowser {
    position: absolute;
    top: 0;
    left: 0;
    width: 200px;
    bottom: 0;
    transition: left 0.2s;
    z-index: 1;
}

.kiwi-container {
    position: absolute;
    top: 0;
    bottom: 40px;
    width: 100%;
}

.kiwi-mediaviewer {
    max-height: 70%;
    overflow: auto;
}

.kiwi-controlinput {
    position: absolute;
    bottom: 0;
    height: 40px;
    width: 100%;
    z-index: 2;
}

/* Small screen will cause the statebrowser to act as a drawer */
@media screen and (max-width: 769px) {
    .kiwi-workspace {
        left: 0;
        margin-left: 0;
    }

    .kiwi-statebrowser {
        left: -200px;
    }

    .kiwi-wrap--statebrowser-drawopen .kiwi-statebrowser {
        left: 0;
    }

    .kiwi-wrap--statebrowser-drawopen .kiwi-workspace {
        left: 75%;
        width: 80%;
    }

    .kiwi-wrap--statebrowser-drawopen .kiwi-workspace::after {
        width: 100%;
        height: 100%;
        opacity: 1;
        z-index: 10;
    }
}
</style>
