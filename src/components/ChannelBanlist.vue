<template>
    <div class="kiwi-channelbanlist">
        <form class="u-form kiwi-channelbanlist" @submit.prevent="">
            <a class="u-link" @click="updateBanlist">{{ $t('bans_refresh') }}</a>

            <table v-if="banlist.length > 0" class="kiwi-channelbanlist-table">
                <tr>
                    <th>{{ $t('bans_user') }}</th>
                    <th>{{ $t('bans_by') }}</th>
                    <th/>
                    <th/>
                </tr>
                <tr v-for="ban in banlist" :key="ban.banned">
                    <td class="kiwi-channelbanlist-table-mask">{{ ban.banned }}</td>
                    <td class="kiwi-channelbanlist-table-bannedby">{{ ban.banned_by }}</td>
                    <td class="kiwi-channelbanlist-table-bannedat">
                        {{ (new Date(ban.banned_at * 1000)).toDateString() }}
                    </td>
                    <td class="kiwi-channelbanlist-table-actions">
                        <i class="fa fa-trash" aria-hidden="true" @click="removeBan(ban.banned)"/>
                    </td>
                </tr>
            </table>
            <div v-else-if="is_refreshing">
                {{ $t('bans_refreshing') }}
            </div>
            <div v-else class="kiwi-channelbanlist-empty">
                {{ $t('bans_nobody') }}
            </div>
        </form>
    </div>
</template>

<script>

export default {
    props: ['buffer'],
    data: function data() {
        return {
            banlist: [],
            is_refreshing: false,
        };
    },
    created: function created() {
        this.updateBanlist();
    },
    methods: {
        updateBanlist: function updateBanlist() {
            if (this.buffer.getNetwork().state !== 'connected' || this.is_refreshing) {
                return;
            }

            let channelName = this.buffer.name;
            this.is_refreshing = true;
            this.buffer.getNetwork().ircClient.banlist(channelName, (banEvent) => {
                this.banlist = banEvent.bans;
                this.is_refreshing = false;
            });
        },
        removeBan: function removeBan(mask) {
            let channelName = this.buffer.name;
            this.buffer.getNetwork().ircClient.unban(channelName, mask);
            this.banlist = this.banlist.filter(ban => ban.banned !== mask);
        },
    },
};
</script>

<style lang="less">
.kiwi-channelbanlist-table {
    width: 100%;
    border-collapse: collapse;
    line-height: 20px;
    margin-top: 10px;
}

.kiwi-channelbanlist-table-bannedat {
    min-width: 150px;
}

.kiwi-channelbanlist-table-actions {
    min-width: 50px;
    text-align: center;
    cursor: pointer;
    position: relative;
    transition: all 0.3s;
    z-index: 1;
}
</style>
