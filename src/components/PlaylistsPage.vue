<template>
    <h2 class="font-bold my-4" v-t="'titles.playlists'" />

    <div class="flex justify-between mb-3">
        <button v-t="'actions.create_playlist'" class="btn" @click="onCreatePlaylist" />
        <div class="flex">
            <button
                v-if="this.playlists.length > 0"
                v-t="'actions.export_to_json'"
                class="btn"
                @click="exportPlaylists"
            />
            <input
                id="fileSelector"
                ref="fileSelector"
                type="file"
                class="display-none"
                @change="importPlaylists"
                multiple="multiple"
            />
            <label for="fileSelector" v-t="'actions.import_from_json'" class="btn ml-2" />
        </div>
    </div>

    <div class="video-grid">
        <div v-for="playlist in playlists" :key="playlist.id">
            <router-link :to="`/playlist?list=${playlist.id}`">
                <img class="w-full" :src="playlist.thumbnail" alt="thumbnail" />
                <div class="relative text-sm">
                    <span
                        class="thumbnail-overlay thumbnail-right"
                        v-text="`${playlist.videos} ${$t('video.videos')}`"
                    />
                </div>
                <p
                    style="display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical"
                    class="my-2 overflow-hidden flex link"
                    :title="playlist.name"
                    v-text="playlist.name"
                />
            </router-link>
            <button class="btn h-auto" @click="showPlaylistEditModal(playlist)" v-t="'actions.edit_playlist'" />
            <button class="btn h-auto ml-2" @click="playlistToDelete = playlist.id" v-t="'actions.delete_playlist'" />
            <ModalComponent v-if="playlist.id == playlistToEdit" @close="playlistToEdit = null">
                <div class="flex flex-col gap-2">
                    <h2 v-t="'actions.edit_playlist'" />
                    <input
                        class="input"
                        type="text"
                        v-model="newPlaylistName"
                        :placeholder="$t('actions.playlist_name')"
                    />
                    <input
                        class="input"
                        type="text"
                        v-model="newPlaylistDescription"
                        :placeholder="$t('actions.playlist_description')"
                    />
                    <button class="btn ml-auto" @click="editPlaylist(playlist)" v-t="'actions.okay'" />
                </div>
            </ModalComponent>
            <ConfirmModal
                v-if="playlistToDelete == playlist.id"
                :message="$t('actions.delete_playlist_confirm')"
                @close="playlistToDelete = null"
                @confirm="onDeletePlaylist(playlist.id)"
            />
        </div>
    </div>
    <hr />

    <h2 class="font-bold my-4" v-t="'titles.bookmarks'" />

    <div v-if="bookmarks" class="video-grid">
        <router-link
            v-for="(playlist, index) in bookmarks"
            :key="playlist.playlistId"
            :to="`/playlist?list=${playlist.playlistId}`"
        >
            <img class="w-full" :src="playlist.thumbnail" alt="thumbnail" />
            <div class="relative text-sm">
                <span class="thumbnail-overlay thumbnail-right" v-text="`${playlist.videos} ${$t('video.videos')}`" />
                <div class="absolute bottom-100px right-5px px-5px z-100" @click.prevent="removeBookmark(index)">
                    <font-awesome-icon class="ml-3" icon="bookmark" />
                </div>
            </div>
            <p
                style="display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical"
                class="my-2 overflow-hidden flex link"
                :title="playlist.name"
                v-text="playlist.name"
            />
            <a :href="playlist.uploaderUrl" class="flex items-center">
                <img class="rounded-full w-32px h-32px" :src="playlist.uploaderAvatar" />
                <span class="ml-3 hover:underline" v-text="playlist.uploader" />
            </a>
        </router-link>
    </div>
    <br />
</template>

<script>
import ConfirmModal from "./ConfirmModal.vue";
import ModalComponent from "./ModalComponent.vue";

export default {
    data() {
        return {
            playlists: [],
            bookmarks: [],
            playlistToDelete: null,
            playlistToEdit: null,
            newPlaylistName: "",
            newPlaylistDescription: "",
        };
    },
    mounted() {
        this.fetchPlaylists();
        this.loadPlaylistBookmarks();
    },
    activated() {
        document.title = this.$t("titles.playlists") + " - Piped";
    },
    methods: {
        fetchPlaylists() {
            this.getPlaylists().then(json => {
                this.playlists = json;
            });
        },
        showPlaylistEditModal(playlist) {
            this.newPlaylistName = playlist.name;
            this.newPlaylistDescription = playlist.description;
            this.playlistToEdit = playlist.id;
        },
        editPlaylist(selectedPlaylist) {
            // save the new name and description since they could be overwritten during the http request
            const newName = this.newPlaylistName;
            const newDescription = this.newPlaylistDescription;
            if (newName != selectedPlaylist.name) {
                this.renamePlaylist(selectedPlaylist.id, newName).then(json => {
                    if (json.error) alert(json.error);
                    else selectedPlaylist.name = newName;
                });
            }
            if (newDescription != selectedPlaylist.description) {
                this.changePlaylistDescription(selectedPlaylist.id, newDescription).then(json => {
                    if (json.error) alert(json.error);
                    else selectedPlaylist.description = newDescription;
                });
            }
            this.playlistToEdit = null;
        },
        onDeletePlaylist(id) {
            this.deletePlaylist(id).then(json => {
                if (json.error) alert(json.error);
                else this.playlists = this.playlists.filter(playlist => playlist.id !== id);
            });
            this.playlistToDelete = null;
        },
        onCreatePlaylist() {
            const name = prompt(this.$t("actions.create_playlist"));
            if (!name) return;
            this.createPlaylist(name).then(json => {
                if (json.error) alert(json.error);
                else this.fetchPlaylists();
            });
        },
        async exportPlaylists() {
            if (!this.playlists) return;
            let json = {
                format: "Piped",
                version: 1,
                playlists: [],
            };
            let tasks = this.playlists.map(playlist => this.fetchPlaylistJson(playlist.id));
            json.playlists = await Promise.all(tasks);
            this.download(JSON.stringify(json), "playlists.json", "application/json");
        },
        async fetchPlaylistJson(playlistId) {
            let playlist = await this.getPlaylist(playlistId);
            return {
                name: playlist.name,
                // possible other types: history, watch later, ...
                type: "playlist",
                // as Invidious supports public and private playlists
                visibility: "private",
                // list of the videos, starting with "https://youtube.com" to clarify that those are YT videos
                videos: playlist.relatedStreams.map(stream => "https://youtube.com" + stream.url),
            };
        },
        async importPlaylists() {
            const files = this.$refs.fileSelector.files;
            for (let file of files) {
                await this.importPlaylistFile(file);
            }
            window.location.reload();
        },
        async importPlaylistFile(file) {
            let text = await file.text();
            let tasks = [];
            // list of playlists exported from Piped
            if (text.includes("playlists")) {
                let playlists = JSON.parse(text).playlists;
                if (!playlists.length) {
                    alert(this.$t("actions.no_valid_playlists"));
                    return;
                }
                for (let playlist of playlists) {
                    tasks.push(this.createPlaylistWithVideos(playlist));
                }
                // CSV from Google Takeout
            } else if (file.name.slice(-4).toLowerCase() == ".csv") {
                const lines = text.split("\n");
                const playlist = {
                    name: lines[1].split(",")[4],
                    videos: lines
                        .slice(4, lines.length)
                        .filter(line => line != "")
                        .map(line => `https://youtube.com/watch?v=${line.split(",")[0]}`),
                };
                tasks.push(this.createPlaylistWithVideos(playlist));
            } else {
                alert(this.$t("actions.no_valid_playlists"));
                return;
            }
            await Promise.all(tasks);
        },
        async createPlaylistWithVideos(playlist) {
            let newPlaylist = await this.createPlaylist(playlist.name);
            let videoIds = playlist.videos.map(url => url.substr(-11));
            await this.addVideosToPlaylist(newPlaylist.playlistId, videoIds);
        },
        async loadPlaylistBookmarks() {
            if (!window.db) return;
            var tx = window.db.transaction("playlist_bookmarks", "readonly");
            var store = tx.objectStore("playlist_bookmarks");
            const cursorRequest = store.openCursor();
            cursorRequest.onsuccess = e => {
                const cursor = e.target.result;
                if (cursor) {
                    this.bookmarks.push(cursor.value);
                    cursor.continue();
                }
            };
        },
        async removeBookmark(index) {
            var tx = window.db.transaction("playlist_bookmarks", "readwrite");
            var store = tx.objectStore("playlist_bookmarks");
            store.delete(this.bookmarks[index].playlistId);
            this.bookmarks.splice(index, 1);
        },
    },
    components: { ConfirmModal, ModalComponent },
};
</script>
