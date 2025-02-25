<template>
    <div v-if="video && isEmbed" class="absolute left-0 top-0 z-50 h-full w-full bg-black">
        <VideoPlayer
            ref="videoPlayer"
            :video="video"
            :sponsors="sponsors"
            :selected-auto-play="false"
            :selected-auto-loop="selectedAutoLoop"
            :is-embed="isEmbed"
        />
    </div>

    <LoadingIndicatorPage :show-content="video && !isEmbed" class="mt-[15rem] w-full">
        <ErrorHandler v-if="video && video.error" :message="video.message" :error="video.error" />
        <Transition>
            <ToastComponent v-if="shouldShowToast" @dismissed="dismiss">
                <i18n-t keypath="info.next_video_countdown">{{ counter }}</i18n-t>
            </ToastComponent>
        </Transition>

        <div v-show="!video.error">
            <div :class="isMobile ? 'flex-col' : 'flex'">
                <keep-alive>
                    <VideoPlayer
                        ref="videoPlayer"
                        :video="video"
                        :sponsors="sponsors"
                        :selected-auto-play="selectedAutoPlay"
                        :selected-auto-loop="selectedAutoLoop"
                        @timeupdate="onTimeUpdate"
                        @ended="onVideoEnded"
                        @navigate-next="navigateNext"
                    />
                </keep-alive>
                <ChaptersBar
                    v-if="video?.chapters?.length > 0 && showChapters"
                    :mobile-layout="isMobile"
                    :chapters="video.chapters"
                    :player-position="currentTime"
                    @seek="navigate"
                />
            </div>
            <!-- video title -->
            <div class="pp-video-title mt-2 break-words text-2xl font-bold" v-text="video.title" />
            <div class="pp-bellow-video mb-3 mt-3 flex flex-wrap">
                <!-- views / date -->
                <div class="flex flex-auto">
                    <span v-t="{ path: 'video.views', args: { views: addCommas(video.views) } }" />
                    <span> · </span>
                    <span v-text="uploadDate" />
                </div>
                <!-- Likes/dilikes -->
                <div class="pp-likes flex children:mr-2">
                    <template v-if="video.likes >= 0">
                        <div class="flex items-center">
                            <div class="i-fa6-solid:thumbs-up" />
                            <strong class="ml-1" v-text="addCommas(video.likes)" />
                        </div>
                        <div class="flex items-center">
                            <div class="i-fa6-solid:thumbs-down" />
                            <strong class="ml-1" v-text="video.dislikes >= 0 ? addCommas(video.dislikes) : '?'" />
                        </div>
                    </template>
                    <template v-if="video.likes < 0">
                        <div>
                            <strong v-t="'video.ratings_disabled'" />
                        </div>
                    </template>
                </div>
            </div>
            <!-- Channel info & options flex container -->
            <div class="pp-watch-bellow-options flex">
                <!-- Channel Image & Info -->
                <div class="flex items-center">
                    <img :src="video.uploaderAvatar" alt="" loading="lazy" />
                    <router-link v-if="video.uploaderUrl" class="link ml-1.5" :to="video.uploaderUrl">{{
                        video.uploader
                    }}</router-link>
                    <!-- Verified Badge -->
                    <i v-if="video.uploaderVerified" class="i-fa6-solid:check ml-1" />
                </div>
                <div class="pp-watch-buttons">
                    <!-- Subscribe button -->
                    <button
                        class="btn"
                        @click="subscribeHandler"
                        v-text="
                            $t('actions.' + (subscribed ? 'unsubscribe' : 'subscribe')) +
                            ' - ' +
                            numberFormat(video.uploaderSubscriberCount)
                        "
                    />
                    <!-- Playlist Add button -->
                    <button class="pp-square btn flex items-center" style="padding: 0" @click="showModal = !showModal">
                        <i class="i-fa6-solid:circle-plus m-0" />
                    </button>
                    <PlaylistAddModal
                        v-if="showModal"
                        :video-id="getVideoId()"
                        :video-info="video"
                        @close="showModal = !showModal"
                    />
                    <!-- Share Dialog -->
                    <ShareModal
                        v-if="showShareModal"
                        :video-id="getVideoId()"
                        :current-time="currentTime"
                        :playlist-id="playlistId"
                        :playlist-index="index"
                        @close="showShareModal = !showShareModal"
                    />
                    <button
                        class="pp-square btn share-btn flex items-center"
                        style="padding: 0"
                        @click="showShareModal = !showShareModal"
                    >
                        <i class="i-fa6-solid:share m-0" />
                    </button>
                    <!-- YouTube -->
                    <WatchOnButton :link="`https://youtu.be/${getVideoId()}`" />
                    <!-- Odysee -->
                    <WatchOnButton v-if="video.lbryId" platform="Odysee" :link="`https://odysee.com/${video.lbryId}`" />
                    <!-- listen / watch toggle -->
                    <router-link
                        :to="toggleListenUrl"
                        role="button"
                        :aria-label="(isListening ? 'Watch ' : 'Listen to ') + video.title"
                        :title="(isListening ? 'Watch ' : 'Listen to ') + video.title"
                        class="pp-square btn flex items-center"
                        style="padding: 0"
                    >
                        <i :class="isListening ? 'i-fa6-solid:tv' : 'i-fa6-solid:headphones'" class="m-0" />
                    </router-link>
                    <!-- RSS Feed button -->
                    <a
                        v-if="video.uploaderUrl"
                        aria-label="RSS feed"
                        title="RSS feed"
                        role="button"
                        :href="`${apiUrl()}/feed/unauthenticated/rss?channels=${video.uploaderUrl.split('/')[2]}`"
                        target="_blank"
                        class="pp-square btn flex items-center"
                        style="padding: 0"
                    >
                        <i class="i-fa6-solid:rss m-0" />
                    </a>
                    <button class="pp-square btn flex items-center" style="padding: 0" @click="downloadCurrentFrame">
                        <i class="i-fa6-solid:download m-0" />
                    </button>
                </div>
            </div>

            <hr class="mb-2" />

            <div
                v-for="metaInfo in video?.metaInfo ?? []"
                :key="metaInfo.title"
                class="btn my-3 flex flex-wrap cursor-default gap-2 px-4 py-2"
            >
                <span>{{ metaInfo.description ?? metaInfo.title }}</span>
                <a v-for="(link, linkIndex) in metaInfo.urls" :key="linkIndex" :href="link" class="underline">{{
                    metaInfo.urlTexts[linkIndex]
                }}</a>
                <br />
            </div>

            <div efy_select class="pp-watch-toggles">
                <input id="showDesc" v-model="showDesc" type="checkbox" />
                <label v-t="'actions.show_description'" for="showDesc" />
                <input id="showComments" v-model="showComments" type="checkbox" @click="toggleComments" />
                <label
                    for="showComments"
                    v-text="`${$t('actions.show_comments')} - ${numberFormat(comments?.commentCount)}`"
                />
                <input id="showRecs" v-model="showRecs" type="checkbox" />
                <label v-t="'actions.show_recommendations'" for="showRecs" />
                <span v-show="video?.chapters?.length > 0">
                    <input id="showChapters" v-model="showChapters" type="checkbox" />
                    <label v-t="'actions.show_chapters'" class="ml-2" for="showChapters" />
                </span>
                <input id="chkAutoLoop" v-model="selectedAutoLoop" type="checkbox" @change="onChange($event)" />
                <label for="chkAutoLoop" v-text="`${$t('actions.loop_this_video')}`" />
                <input id="chkAutoPlay" v-model="selectedAutoPlay" type="checkbox" @change="onChange($event)" />
                <label for="chkAutoPlay" v-text="`${$t('actions.auto_play_next_video')}`" />
            </div>

            <template v-if="showDesc">
                <hr />
                <!-- eslint-disable-next-line vue/no-v-html -->
                <div class="description break-words" v-html="purifiedDescription" />
                <hr />
                <div
                    v-if="sponsors && sponsors.segments"
                    v-text="`${$t('video.sponsor_segments')}: ${sponsors.segments.length}`"
                />
                <div v-if="video.category" v-text="`${$t('video.category')}: ${video.category}`" />
                <div v-text="`${$t('video.license')}: ${video.license}`" />
                <div class="capitalize" v-text="`${$t('video.visibility')}: ${video.visibility}`" />
                <hr />
                <div v-if="video.tags" class="video-tags">
                    <router-link
                        v-for="tag in video.tags"
                        :key="tag"
                        class="efy_trans_filter efy_shadow_trans line-clamp-1"
                        :to="`/results?search_query=${encodeURIComponent(tag)}`"
                        >{{ tag }}</router-link
                    >
                </div>
            </template>
        </div>

        <hr />

        <div class="pp-rec-vids grid">
            <div v-if="!showComments" class="w-full"></div>
            <div v-else-if="!comments">
                <p v-t="'comment.loading'" class="mt-8 text-center"></p>
            </div>
            <div v-else-if="comments.disabled">
                <p v-t="'comment.disabled'" class="mt-8 text-center"></p>
            </div>
            <div v-else ref="comments" class="pp-comments">
                <CommentItem
                    v-for="comment in comments.comments"
                    :key="comment.commentId"
                    :comment="comment"
                    :uploader="video.uploader"
                    :uploader-avatar-url="video.uploaderAvatar"
                    :video-id="getVideoId()"
                    class="efy_trans_filter efy_shadow_trans"
                />
            </div>

            <div v-if="video" class="order-first sm:order-last">
                <PlaylistVideos
                    v-if="playlist"
                    :playlist-id="playlistId"
                    :playlist="playlist"
                    :selected-index="index"
                    :prefer-listen="isListening"
                />
                <div v-show="showRecs" class="pp-show-recs">
                    <h6 efy_card style="padding: 5rem 10rem 3rem; margin: 0">Recommended</h6>
                    <ContentItem
                        v-for="related in video.relatedStreams"
                        :key="related.url"
                        :item="related"
                        :prefer-listen="isListening"
                        class="mb-4"
                        height="94"
                        width="168"
                    />
                </div>
            </div>
        </div>
    </LoadingIndicatorPage>
</template>

<script>
import VideoPlayer from "./VideoPlayer.vue";
import ContentItem from "./ContentItem.vue";
import ErrorHandler from "./ErrorHandler.vue";
import CommentItem from "./CommentItem.vue";
import ChaptersBar from "./ChaptersBar.vue";
import PlaylistAddModal from "./PlaylistAddModal.vue";
import ShareModal from "./ShareModal.vue";
import PlaylistVideos from "./PlaylistVideos.vue";
import WatchOnButton from "./WatchOnButton.vue";
import LoadingIndicatorPage from "./LoadingIndicatorPage.vue";
import ToastComponent from "./ToastComponent.vue";
import { parseTimeParam } from "@/utils/Misc";
import { purifyHTML, rewriteDescription } from "@/utils/HtmlUtils";

export default {
    name: "App",
    components: {
        VideoPlayer,
        ContentItem,
        ErrorHandler,
        CommentItem,
        ChaptersBar,
        PlaylistAddModal,
        ShareModal,
        PlaylistVideos,
        WatchOnButton,
        LoadingIndicatorPage,
        ToastComponent,
    },
    data() {
        const smallViewQuery = window.matchMedia("(max-width: 640px)");
        return {
            video: null,
            playlistId: null,
            playlist: null,
            index: null,
            sponsors: null,
            selectedAutoLoop: false,
            selectedAutoPlay: null,
            showComments: true,
            showDesc: false,
            showRecs: true,
            showChapters: true,
            comments: null,
            subscribed: false,
            channelId: null,
            active: true,
            smallViewQuery: smallViewQuery,
            smallView: smallViewQuery.matches,
            showModal: false,
            showShareModal: false,
            isMobile: true,
            currentTime: 0,
            shouldShowToast: false,
            timeoutCounter: null,
            counter: 0,
        };
    },
    computed: {
        isListening(_this) {
            return _this.getPreferenceBoolean("listen", false);
        },
        toggleListenUrl(_this) {
            const url = new URL(window.location.href);
            url.searchParams.set("listen", _this.isListening ? "0" : "1");
            return url.pathname + url.search;
        },
        isEmbed(_this) {
            return String(_this.$route.path).indexOf("/embed/") == 0;
        },
        uploadDate(_this) {
            return new Date(_this.video.uploadDate).toLocaleString(undefined, {
                month: "short",
                day: "numeric",
                year: "numeric",
            });
        },
        defaultCounter(_this) {
            return _this.getPreferenceNumber("autoPlayNextCountdown", 5);
        },
        purifiedDescription() {
            return purifyHTML(this.video.description);
        },
    },
    mounted() {
        // check screen size
        if (window.innerWidth >= 1024) {
            this.isMobile = false;
        }
        // add an event listener to watch for screen size changes
        window.addEventListener("resize", () => {
            if (window.innerWidth >= 1024) {
                this.isMobile = false;
            } else {
                this.isMobile = true;
            }
        });
        this.getVideoData().then(() => {
            (async () => {
                const videoId = this.getVideoId();
                const instance = this;
                if (window.db && this.getPreferenceBoolean("watchHistory", false) && !this.video.error) {
                    var tx = window.db.transaction("watch_history", "readwrite");
                    var store = tx.objectStore("watch_history");
                    var request = store.get(videoId);
                    request.onsuccess = function (event) {
                        var video = event.target.result;
                        if (video) {
                            video.watchedAt = Date.now();
                        } else {
                            video = {
                                videoId: videoId,
                                title: instance.video.title,
                                duration: instance.video.duration,
                                thumbnail: instance.video.thumbnailUrl,
                                uploaderUrl: instance.video.uploaderUrl,
                                uploaderName: instance.video.uploader,
                                watchedAt: Date.now(),
                            };
                        }
                        store.put(video);
                    };
                }
            })();
            if (this.active) this.$refs.videoPlayer.loadVideo();
        });
        this.playlistId = this.$route.query.list;
        this.index = Number(this.$route.query.index);
        this.getPlaylistData();
        this.getSponsors();
        if (!this.isEmbed && this.showComments) this.getComments();
        if (this.isEmbed) document.querySelector("html").style.overflow = "hidden";
        window.addEventListener("click", this.handleClick);
        window.addEventListener("resize", () => {
            this.smallView = this.smallViewQuery.matches;
        });
    },
    activated() {
        this.active = true;
        this.selectedAutoPlay = this.getPreferenceBoolean("autoplay", false);
        this.showComments = !this.getPreferenceBoolean("minimizeComments", false);
        this.showDesc = !this.getPreferenceBoolean("minimizeDescription", true);
        this.showRecs = !this.getPreferenceBoolean("minimizeRecommendations", false);
        this.showChapters = !this.getPreferenceBoolean("minimizeChapters", false);
        if (this.video?.duration) {
            document.title = this.video.title + " - Piped";
            this.$refs.videoPlayer.loadVideo();
        }
        window.addEventListener("scroll", this.handleScroll);
    },
    deactivated() {
        this.active = false;
        window.removeEventListener("scroll", this.handleScroll);
        this.dismiss();
    },
    unmounted() {
        window.removeEventListener("scroll", this.handleScroll);
        window.removeEventListener("click", this.handleClick);
        this.dismiss();
    },
    methods: {
        fetchVideo() {
            return this.fetchJson(this.apiUrl() + "/streams/" + this.getVideoId());
        },
        async fetchSponsors() {
            var selectedSkip = this.getPreferenceString(
                "selectedSkip",
                "sponsor,interaction,selfpromo,music_offtopic",
            ).split(",");
            const skipOptions = this.getPreferenceJSON("skipOptions");
            if (skipOptions !== undefined) {
                selectedSkip = Object.keys(skipOptions).filter(
                    k => skipOptions[k] !== undefined && skipOptions[k] !== "no",
                );
            }

            const sponsors = await this.fetchJson(this.apiUrl() + "/sponsors/" + this.getVideoId(), {
                category: JSON.stringify(selectedSkip),
            });

            sponsors?.segments?.forEach(segment => {
                const option = skipOptions?.[segment.category];
                segment.autoskip = option === undefined || option === "auto";
            });

            const minSegmentLength = Math.max(this.getPreferenceNumber("minSegmentLength", 0), 0);
            sponsors.segments = sponsors.segments?.filter(segment => {
                const length = segment.segment[1] - segment.segment[0];
                return length >= minSegmentLength;
            });

            return sponsors;
        },
        toggleComments() {
            this.showComments = !this.showComments;
            if (this.showComments && this.comments === null) {
                this.fetchComments();
            }
        },
        fetchComments() {
            return this.fetchJson(this.apiUrl() + "/comments/" + this.getVideoId());
        },
        onChange() {
            this.setPreference("autoplay", this.selectedAutoPlay, true);
        },
        async getVideoData() {
            await this.fetchVideo()
                .then(data => {
                    this.video = data;
                    this.video.id = this.getVideoId();
                })
                .then(() => {
                    if (!this.video.error) {
                        document.title = this.video.title + " - Piped";
                        this.channelId = this.video.uploaderUrl.split("/")[2];
                        if (!this.isEmbed) this.fetchSubscribedStatus();

                        const parser = new DOMParser();
                        const xmlDoc = parser.parseFromString(this.video.description, "text/html");
                        xmlDoc.querySelectorAll("a").forEach(elem => {
                            if (!elem.innerText.match(/(?:[\d]{1,2}:)?(?:[\d]{1,2}):(?:[\d]{1,2})/))
                                elem.outerHTML = elem.getAttribute("href");
                        });
                        xmlDoc.querySelectorAll("br").forEach(elem => (elem.outerHTML = "\n"));
                        this.video.description = rewriteDescription(xmlDoc.querySelector("body").innerHTML);
                        this.updateWatched(this.video.relatedStreams);

                        this.fetchDeArrowContent(this.video.relatedStreams);
                    }
                });
        },
        async getPlaylistData() {
            if (this.playlistId) {
                this.playlist = await this.getPlaylist(this.playlistId);
                await this.fetchPlaylistPages().then(() => {
                    if (!(this.index >= 0)) {
                        for (let i = 0; i < this.playlist.relatedStreams.length; i++)
                            if (this.playlist.relatedStreams[i].url.substr(-11) == this.getVideoId()) {
                                this.index = i + 1;
                                this.$router.replace({
                                    query: { ...this.$route.query, index: this.index },
                                });
                                break;
                            }
                    }
                });
                await this.fetchPlaylistPages().then(() => {
                    this.fetchDeArrowContent(this.playlist.relatedStreams);
                });
            }
        },
        async fetchPlaylistPages() {
            if (this.playlist.nextpage) {
                await this.fetchJson(this.apiUrl() + "/nextpage/playlists/" + this.playlistId, {
                    nextpage: this.playlist.nextpage,
                }).then(json => {
                    this.playlist.relatedStreams = this.playlist.relatedStreams.concat(json.relatedStreams);
                    this.playlist.nextpage = json.nextpage;
                });
                await this.fetchPlaylistPages();
            }
        },
        async getSponsors() {
            if (this.getPreferenceBoolean("sponsorblock", true))
                this.fetchSponsors().then(data => (this.sponsors = data));
        },
        async getComments() {
            this.comments = await this.fetchComments();
        },
        async fetchSubscribedStatus() {
            if (!this.channelId) return;

            this.subscribed = await this.fetchSubscriptionStatus(this.channelId);
        },
        subscribeHandler() {
            this.toggleSubscriptionState(this.channelId, this.subscribed).then(success => {
                if (success) this.subscribed = !this.subscribed;
            });
        },
        handleClick(event) {
            if (!event || !event.target) return;
            if (!event.target.matches("a[href]")) return;
            const target = event.target;
            if (!target.getAttribute("href")) return;
            if (this.handleTimestampLinks(target)) {
                event.preventDefault();
            }
        },
        handleTimestampLinks(target) {
            try {
                const url = new URL(target.getAttribute("href"), document.baseURI);
                if (
                    url.searchParams.size > 2 ||
                    url.searchParams.get("v") !== this.getVideoId() ||
                    !url.searchParams.has("t")
                ) {
                    return false;
                }
                const time = parseTimeParam(url.searchParams.get("t"));
                if (time) {
                    this.navigate(time);
                }
                return true;
            } catch (e) {
                console.error(e);
            }
            return false;
        },
        handleScroll() {
            if (this.loading || !this.comments || !this.comments.nextpage) return;
            if (window.innerHeight + window.scrollY >= this.$refs.comments?.offsetHeight - window.innerHeight) {
                this.loading = true;
                this.fetchJson(this.apiUrl() + "/nextpage/comments/" + this.getVideoId(), {
                    nextpage: this.comments.nextpage,
                }).then(json => {
                    this.comments.nextpage = json.nextpage;
                    this.loading = false;
                    this.comments.comments = this.comments.comments.concat(json.comments);
                });
            }
        },
        getVideoId() {
            if (this.$route.query.video_ids) {
                const videos_list = this.$route.query.video_ids.split(",");
                this.index = Number(this.$route.query.index ?? 0);
                return videos_list[this.index];
            }

            return this.$route.query.v || this.$route.params.v;
        },
        navigate(time) {
            this.$refs.videoPlayer.seek(time);
        },
        onTimeUpdate(time) {
            this.currentTime = time;
        },
        onVideoEnded() {
            if (
                !this.selectedAutoLoop &&
                this.selectedAutoPlay &&
                (this.playlist?.relatedStreams?.length > 0 || this.video.relatedStreams.length > 0)
            ) {
                this.showToast();
            }
        },
        showToast() {
            this.counter = this.defaultCounter;
            if (this.counter < 1) {
                this.navigateNext();
                return;
            }
            if (this.timeoutCounter) clearInterval(this.timeoutCounter);
            this.timeoutCounter = setInterval(() => {
                this.counter--;
                if (this.counter === 0) {
                    this.dismiss();
                    this.navigateNext();
                }
            }, 1000);
            this.shouldShowToast = true;
        },
        dismiss() {
            clearInterval(this.timeoutCounter);
            this.shouldShowToast = false;
        },
        navigateNext() {
            const params = this.$route.query;
            const video_ids = this.$route.query.video_ids?.split(",") ?? [];
            let url;
            if (this.playlist) {
                url = this.playlist?.relatedStreams?.[this.index]?.url ?? this.video.relatedStreams[0].url;
            } else if (video_ids.length > this.index + 1) {
                url = `${this.$route.path}?index=${this.index + 1}`;
            } else {
                url = this.video.relatedStreams[0].url;
            }
            const searchParams = new URLSearchParams();
            for (var param in params)
                switch (param) {
                    case "v":
                    case "t":
                        break;
                    case "index":
                        if (this.playlist && this.index < this.playlist.relatedStreams.length)
                            searchParams.set("index", this.index + 1);
                        break;
                    case "list":
                        if (this.index < this.playlist.relatedStreams.length) searchParams.set("list", params.list);
                        break;
                    default:
                        searchParams.set(param, params[param]);
                        break;
                }
            // save the fullscreen state
            searchParams.set("fullscreen", this.$refs.videoPlayer.$ui.getControls().isFullScreenEnabled());
            const paramStr = searchParams.toString();
            if (paramStr.length > 0) url += "&" + paramStr;
            this.$router.push(url);
        },
        downloadCurrentFrame() {
            const video = document.querySelector("video");
            const canvas = document.createElement("canvas");
            canvas.width = video.videoWidth;
            canvas.height = video.videoHeight;

            const context = canvas.getContext("2d");
            context.drawImage(video, 0, 0, canvas.width, canvas.height);

            let link = document.createElement("a");
            const currentTime = Math.round(video.currentTime * 1000) / 1000;
            link.download = `${this.video.title}_${currentTime}s.png`;
            link.href = canvas.toDataURL();
            link.click();
        },
    },
};
</script>

<style>
.v-enter-from,
.v-leave-to {
    opacity: 0;
    transform: translateX(100%) scale(0.5);
}
.description a {
    text-decoration: underline;
    filter: brightness(0.75);
}
.pp-watch-toggles {
    display: flex;
    flex-wrap: wrap;
    gap: var(--efy_gap0);
    & label {
        margin: 0;
    }
}
video::-webkit-media-text-track-display {
    display: flex;
    width: fit-content !important;
    position: relative !important;
    top: calc(100% - 150rem) !important;
    height: fit-content !important;
    padding: 2rem 8rem;
    margin: auto;
    margin-bottom: 8rem;
    background: #0008 !important;
    backdrop-filter: blur(20rem);
    color: #fff;
    border-radius: var(--efy_radius);
    font-family: var(--efy_font_family);
    font-size: 22rem !important;
}
video::cue {
    background: transparent;
}
.player-container {
    &.pp-trans video::-webkit-media-text-track-display {
        background: transparent !important;
        backdrop-filter: none;
        margin-bottom: unset;
        line-height: 1.2;
        text-shadow: 0 0 5rem #000;
    }
    &.pp-solid video::-webkit-media-text-track-display {
        background: var(--efy_bg) !important;
        color: var(--efy_text);
        backdrop-filter: none;
    }
}

@media (max-width: 639px) {
    video::-webkit-media-text-track-display {
        font-size: 16rem !important;
        top: calc(100% - 120rem) !important;
    }
}
</style>
