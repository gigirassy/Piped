<template>
    <div
        ref="container"
        data-shaka-player-container
        class="efy_trans_filter_off relative max-h-screen w-full flex justify-center"
        :class="{ 'player-container': !isEmbed }"
    >
        <video ref="videoEl" class="w-full" data-shaka-player :autoplay="shouldAutoPlay" :loop="selectedAutoLoop" />
        <span
            id="preview-container"
            ref="previewContainer"
            class="absolute bottom-0 z-[2000] mb-[3.5%] hidden flex-col items-center"
        >
            <canvas id="preview" ref="preview" style="border-radius: var(--efy_radius0)" />
            <span
                class="w-min"
                style="
                    border-radius: var(--efy_radius0);
                    background: var(--efy_text2);
                    color: var(--efy_text);
                    padding: 3rem 6rem;
                "
                v-text="timeFormat(currentTime)"
            />
        </span>
        <button
            v-if="inSegment"
            class="skip-segment-button"
            type="button"
            :aria-label="$t('actions.skip_segment')"
            aria-pressed="false"
            @click="onClickSkipSegment"
        >
            <span v-t="'actions.skip_segment'" />
            <i class="material-icons-round">skip_next</i>
        </button>
        <span
            v-if="error > 0"
            v-t="{ path: 'player.failed', args: [error] }"
            class="absolute top-8 rounded bg-black/80 p-2 text-lg backdrop-blur-sm"
        />
    </div>

    <ModalComponent v-if="showSpeedModal" @close="showSpeedModal = false">
        <h2 v-t="'actions.playback_speed'" />
        <div class="flex flex-col">
            <input
                v-model="playbackSpeedInput"
                class="input my-3"
                type="text"
                :placeholder="$t('actions.playback_speed')"
                @keyup.enter="setSpeedFromInput()"
            />
            <button v-t="'actions.okay'" class="btn ml-auto w-min" @click="setSpeedFromInput()" />
        </div>
    </ModalComponent>
</template>

<script>
import "shaka-player/dist/controls.css";
import { parseTimeParam } from "@/utils/Misc";
import ModalComponent from "./ModalComponent.vue";

const shaka = import("shaka-player/dist/shaka-player.ui.js");
const hotkeys = import("hotkeys-js");

export default {
    components: { ModalComponent },
    props: {
        video: {
            type: Object,
            default: () => {
                return {};
            },
        },
        sponsors: {
            type: Object,
            default: () => {
                return {};
            },
        },
        selectedAutoPlay: Boolean,
        selectedAutoLoop: Boolean,
        isEmbed: Boolean,
    },
    emits: ["timeupdate", "ended", "navigateNext"],
    data() {
        return {
            lastUpdate: new Date().getTime(),
            initialSeekComplete: false,
            destroying: false,
            inSegment: false,
            isHoveringTimebar: false,
            showSpeedModal: false,
            playbackSpeedInput: null,
            currentTime: 0,
            seekbarPadding: 2,
            error: 0,
        };
    },
    computed: {
        shouldAutoPlay: _this => {
            return _this.getPreferenceBoolean("playerAutoPlay", true) && !_this.isEmbed;
        },
        preferredVideoCodecs: _this => {
            var preferredVideoCodecs = [];
            const enabledCodecs = _this.getPreferenceString("enabledCodecs", "vp9,avc").split(",");

            if (
                _this.$refs.videoEl.canPlayType('video/mp4; codecs="av01.0.08M.08"') !== "" &&
                enabledCodecs.includes("av1")
            )
                preferredVideoCodecs.push("av01");
            if (_this.$refs.videoEl.canPlayType('video/webm; codecs="vp9"') !== "" && enabledCodecs.includes("vp9"))
                preferredVideoCodecs.push("vp9");
            if (
                _this.$refs.videoEl.canPlayType('video/mp4; codecs="avc1.4d401f"') !== "" &&
                enabledCodecs.includes("avc")
            )
                preferredVideoCodecs.push("avc1");

            return preferredVideoCodecs;
        },
    },
    mounted() {
        if (!this.$shaka) this.shakaPromise = shaka.then(shaka => shaka.default).then(shaka => (this.$shaka = shaka));
        if (!this.$hotkeys)
            this.hotkeysPromise = hotkeys.then(mod => mod.default).then(hotkeys => (this.$hotkeys = hotkeys));
    },
    activated() {
        this.destroying = false;
        this.sponsors?.segments?.forEach(segment => (segment.skipped = false));
        this.hotkeysPromise.then(() => {
            var self = this;
            this.$hotkeys(
                "f,m,j,k,l,c,space,up,down,left,right,0,1,2,3,4,5,6,7,8,9,shift+n,shift+s,shift+,,shift+.,alt+p,return,.,,",
                function (e, handler) {
                    const videoEl = self.$refs.videoEl;
                    switch (handler.key) {
                        case "f":
                            self.$ui.getControls().toggleFullScreen();
                            e.preventDefault();
                            break;
                        case "m":
                            videoEl.muted = !videoEl.muted;
                            e.preventDefault();
                            break;
                        case "j":
                            videoEl.currentTime = Math.max(videoEl.currentTime - 15, 0);
                            e.preventDefault();
                            break;
                        case "l":
                            videoEl.currentTime = videoEl.currentTime + 15;
                            e.preventDefault();
                            break;
                        case "c":
                            self.$player.setTextTrackVisibility(!self.$player.isTextTrackVisible());
                            e.preventDefault();
                            break;
                        case "k":
                        case "space":
                            if (videoEl.paused) videoEl.play();
                            else videoEl.pause();
                            e.preventDefault();
                            break;
                        case "up":
                            videoEl.volume = Math.min(videoEl.volume + 0.05, 1);
                            e.preventDefault();
                            break;
                        case "down":
                            videoEl.volume = Math.max(videoEl.volume - 0.05, 0);
                            e.preventDefault();
                            break;
                        case "left":
                            videoEl.currentTime = Math.max(videoEl.currentTime - 5, 0);
                            e.preventDefault();
                            break;
                        case "right":
                            videoEl.currentTime = videoEl.currentTime + 5;
                            e.preventDefault();
                            break;
                        case "0":
                            videoEl.currentTime = 0;
                            e.preventDefault();
                            break;
                        case "1":
                            videoEl.currentTime = videoEl.duration * 0.1;
                            e.preventDefault();
                            break;
                        case "2":
                            videoEl.currentTime = videoEl.duration * 0.2;
                            e.preventDefault();
                            break;
                        case "3":
                            videoEl.currentTime = videoEl.duration * 0.3;
                            e.preventDefault();
                            break;
                        case "4":
                            videoEl.currentTime = videoEl.duration * 0.4;
                            e.preventDefault();
                            break;
                        case "5":
                            videoEl.currentTime = videoEl.duration * 0.5;
                            e.preventDefault();
                            break;
                        case "6":
                            videoEl.currentTime = videoEl.duration * 0.6;
                            e.preventDefault();
                            break;
                        case "7":
                            videoEl.currentTime = videoEl.duration * 0.7;
                            e.preventDefault();
                            break;
                        case "8":
                            videoEl.currentTime = videoEl.duration * 0.8;
                            e.preventDefault();
                            break;
                        case "9":
                            videoEl.currentTime = videoEl.duration * 0.9;
                            e.preventDefault();
                            break;
                        case "shift+n":
                            self.$emit("navigateNext");
                            e.preventDefault();
                            break;
                        case "shift+s":
                            self.showSpeedModal = true;
                            break;
                        case "shift+,":
                            self.adjustPlaybackSpeed(videoEl.playbackRate - 0.25);
                            break;
                        case "shift+.":
                            self.adjustPlaybackSpeed(videoEl.playbackRate + 0.25);
                            break;
                        case "alt+p":
                            document.pictureInPictureElement
                                ? document.exitPictureInPicture()
                                : videoEl.requestPictureInPicture();
                            break;
                        case "return":
                            self.skipSegment(videoEl);
                            break;
                        case ".":
                            videoEl.currentTime += 0.04;
                            e.preventDefault();
                            break;
                        case ",":
                            videoEl.currentTime -= 0.04;
                            e.preventDefault();
                            break;
                    }
                },
            );
        });
    },
    deactivated() {
        this.destroying = true;
        this.destroy(true);
    },
    unmounted() {
        this.destroying = true;
        this.destroy(true);
    },
    methods: {
        async loadVideo() {
            this.updateSponsors();

            const component = this;
            const videoEl = this.$refs.videoEl;

            videoEl.setAttribute("poster", this.video.thumbnailUrl);

            const noPrevPlayer = !this.$player;

            var streams = [];

            streams.push(...this.video.audioStreams);
            streams.push(...this.video.videoStreams);

            const MseSupport = window.MediaSource !== undefined;

            const lbry = null;

            var uri;
            var mime;

            if (this.video.livestream) {
                uri = this.video.hls;
                mime = "application/x-mpegURL";
            } else if (this.video.audioStreams.length > 0 && !lbry && MseSupport) {
                if (!this.video.dash) {
                    const dash = (await import("../utils/DashUtils.js")).generate_dash_file_from_formats(
                        streams,
                        this.video.duration,
                    );

                    uri = "data:application/dash+xml;charset=utf-8;base64," + btoa(dash);
                } else {
                    const url = new URL(this.video.dash);
                    url.searchParams.set("rewrite", false);
                    uri = url.toString();
                }
                mime = "application/dash+xml";
            } else if (lbry) {
                uri = lbry.url;
                if (this.getPreferenceBoolean("proxyLBRY", false)) {
                    const url = new URL(uri);
                    const proxyURL = new URL(this.video.proxyUrl);
                    let proxyPath = proxyURL.pathname;
                    if (proxyPath.lastIndexOf("/") === proxyPath.length - 1) {
                        proxyPath = proxyPath.substring(0, proxyPath.length - 1);
                    }

                    url.searchParams.set("host", url.host);
                    url.protocol = proxyURL.protocol;
                    url.host = proxyURL.host;
                    url.pathname = proxyPath + url.pathname;
                    uri = url.toString();
                }
                const contentType = await fetch(uri, {
                    method: "HEAD",
                }).then(response => {
                    uri = response.url;
                    return response.headers.get("Content-Type");
                });
                mime = contentType;
            } else if (this.video.hls) {
                uri = this.video.hls;
                mime = "application/x-mpegURL";
            } else {
                uri = this.video.videoStreams.findLast(stream => stream.codec == null).url;
                mime = "video/mp4";
            }

            if (noPrevPlayer)
                this.shakaPromise.then(async () => {
                    if (this.destroying) return;
                    this.$shaka.polyfill.installAll();

                    const localPlayer = new this.$shaka.Player();
                    await localPlayer.attach(videoEl);
                    const proxyURL = new URL(component.video.proxyUrl);
                    let proxyPath = proxyURL.pathname;
                    if (proxyPath.lastIndexOf("/") === proxyPath.length - 1) {
                        proxyPath = proxyPath.substring(0, proxyPath.length - 1);
                    }

                    localPlayer.getNetworkingEngine().registerRequestFilter((_type, request) => {
                        const uri = request.uris[0];
                        var url = new URL(uri);
                        const headers = request.headers;
                        if (
                            url.host.endsWith(".googlevideo.com") ||
                            (url.host.endsWith(".lbryplayer.xyz") &&
                                (component.getPreferenceBoolean("proxyLBRY", false) || headers.Range))
                        ) {
                            url.searchParams.set("host", url.host);
                            url.protocol = proxyURL.protocol;
                            url.host = proxyURL.host;
                            url.pathname = proxyPath + url.pathname;
                            request.uris[0] = url.toString();
                        }
                        if (url.pathname === proxyPath + "/videoplayback") {
                            if (headers.Range) {
                                url.searchParams.set("range", headers.Range.split("=")[1]);
                                request.headers = {};
                                request.uris[0] = url.toString();
                            }
                        }
                    });

                    localPlayer.configure(
                        "streaming.bufferingGoal",
                        Math.max(this.getPreferenceNumber("bufferGoal", 10), 10),
                    );

                    this.setPlayerAttrs(localPlayer, videoEl, uri, mime, this.$shaka);
                });
            else this.setPlayerAttrs(this.$player, videoEl, uri, mime, this.$shaka);

            if (noPrevPlayer) {
                videoEl.addEventListener("loadeddata", () => {
                    if (document.pictureInPictureElement) videoEl.requestPictureInPicture();
                });
                videoEl.addEventListener("timeupdate", () => {
                    const time = videoEl.currentTime;
                    this.$emit("timeupdate", time);
                    this.updateProgressDatabase(time);
                    if (this.sponsors && this.sponsors.segments) {
                        const segment = this.findCurrentSegment(time);
                        this.inSegment = !!segment;
                        if (segment?.autoskip && (!segment.skipped || this.selectedAutoLoop)) {
                            this.skipSegment(videoEl, segment);
                        }
                    }
                });

                videoEl.addEventListener("volumechange", () => {
                    this.setPreference("volume", videoEl.volume, true);
                });

                videoEl.addEventListener("ratechange", e => {
                    const rate = videoEl.playbackRate;
                    if (rate > 0 && !isNaN(videoEl.duration) && !isNaN(videoEl.duration - e.timeStamp / 1000))
                        this.setPreference("rate", rate, true);
                });

                videoEl.addEventListener("ended", () => {
                    this.$emit("ended");
                });
            }

            //TODO: Add sponsors on seekbar: https://github.com/ajayyy/SponsorBlock/blob/e39de9fd852adb9196e0358ed827ad38d9933e29/src/js-components/previewBar.ts#L12
        },
        findCurrentSegment(time) {
            return this.sponsors?.segments?.find(s => time >= s.segment[0] && time < s.segment[1]);
        },
        onClickSkipSegment() {
            const videoEl = this.$refs.videoEl;
            this.skipSegment(videoEl);
        },
        skipSegment(videoEl, segment) {
            const time = videoEl.currentTime;
            if (!segment) segment = this.findCurrentSegment(time);
            if (!segment) return;
            console.log("Skipped segment at " + time);
            videoEl.currentTime = segment.segment[1];
            segment.skipped = true;
        },
        async setPlayerAttrs(localPlayer, videoEl, uri, mime, shaka) {
            const url = "/watch?v=" + this.video.id;

            if (!this.$ui) {
                this.destroy();
                const OpenButton = class extends shaka.ui.Element {
                    constructor(parent, controls) {
                        super(parent, controls);

                        this.newTabButton_ = document.createElement("button");
                        this.newTabButton_.classList.add("shaka-cast-button");
                        this.newTabButton_.classList.add("shaka-tooltip");
                        this.newTabButton_.ariaPressed = "false";

                        this.newTabIcon_ = document.createElement("i");
                        this.newTabIcon_.classList.add("material-icons-round");
                        this.newTabIcon_.textContent = "launch";
                        this.newTabButton_.appendChild(this.newTabIcon_);

                        const label = document.createElement("label");
                        label.classList.add("shaka-overflow-button-label");
                        label.classList.add("shaka-overflow-menu-only");
                        this.newTabNameSpan_ = document.createElement("span");
                        this.newTabNameSpan_.innerText = "Open in new tab";
                        label.appendChild(this.newTabNameSpan_);

                        this.newTabButton_.appendChild(label);
                        this.parent.appendChild(this.newTabButton_);

                        this.eventManager.listen(this.newTabButton_, "click", () => {
                            this.video.pause();
                            window.open(url);
                        });
                    }
                };

                OpenButton.Factory = class {
                    create(rootElement, controls) {
                        return new OpenButton(rootElement, controls);
                    }
                };

                shaka.ui.OverflowMenu.registerElement("open_new_tab", new OpenButton.Factory());

                this.$ui = new shaka.ui.Overlay(localPlayer, this.$refs.container, videoEl);

                const overflowMenuButtons = ["quality", "captions", "picture_in_picture", "playback_rate", "airplay"];

                if (this.isEmbed) {
                    overflowMenuButtons.push("open_new_tab");
                }

                const config = {
                    overflowMenuButtons: overflowMenuButtons,
                    seekBarColors: {
                        base: "rgba(255, 255, 255, 0.3)",
                        buffered: "rgba(255, 255, 255, 0.54)",
                        played: "var(--efy_piped_color)",
                    },
                };

                this.$ui.configure(config);
            }

            this.updateMarkers();

            const event = new Event("playerInit");
            window.dispatchEvent(event);

            const player = this.$ui.getControls().getPlayer();

            this.setupSeekbarPreview();

            this.$player = player;

            const disableVideo = this.getPreferenceBoolean("listen", false) && !this.video.livestream;

            const prefetchLimit = Math.min(Math.max(this.getPreferenceNumber("prefetchLimit", 2), 0), 10);

            this.$player.configure({
                preferredVideoCodecs: this.preferredVideoCodecs,
                preferredAudioCodecs: ["opus", "mp4a"],
                manifest: {
                    disableVideo: disableVideo,
                },
                streaming: {
                    segmentPrefetchLimit: prefetchLimit,
                    retryParameters: {
                        maxAttempts: Infinity,
                        baseDelay: 250,
                        backoffFactor: 1.5,
                    },
                },
            });

            const quality = this.getPreferenceNumber("quality", 0);
            const qualityConds =
                quality > 0 && (this.video.audioStreams.length > 0 || this.video.livestream) && !disableVideo;
            if (qualityConds) this.$player.configure("abr.enabled", false);

            const time = this.$route.query.t ?? this.$route.query.start;

            var startTime = 0;

            if (time) {
                startTime = parseTimeParam(time);
                this.initialSeekComplete = true;
            } else if (window.db && this.getPreferenceBoolean("watchHistory", false)) {
                await new Promise(resolve => {
                    var tx = window.db.transaction("watch_history", "readonly");
                    var store = tx.objectStore("watch_history");
                    var request = store.get(this.video.id);
                    request.onsuccess = function (event) {
                        var video = event.target.result;
                        const currentTime = video?.currentTime;
                        if (currentTime) {
                            if (currentTime < video.duration * 0.9) {
                                startTime = currentTime;
                            }
                        }
                        resolve();
                    };

                    tx.oncomplete = () => {
                        this.initialSeekComplete = true;
                    };
                });
            } else {
                this.initialSeekComplete = true;
            }

            player
                .load(uri, startTime, mime)
                .then(() => {
                    const isSafari = window.navigator?.vendor?.includes("Apple");

                    if (!isSafari) {
                        // Set the audio language
                        const prefLang = this.getPreferenceString("hl", "en").substr(0, 2);
                        var lang = "en";
                        for (var l in player.getAudioLanguages()) {
                            if (l == prefLang) {
                                lang = l;
                                return;
                            }
                        }
                        player.selectAudioLanguage(lang);
                    }

                    const audioLanguages = player.getAudioLanguages();
                    if (audioLanguages.length > 1) {
                        const overflowMenuButtons = this.$ui.getConfiguration().overflowMenuButtons;
                        // append language menu on index 1
                        const newOverflowMenuButtons = [
                            ...overflowMenuButtons.slice(0, 1),
                            "language",
                            ...overflowMenuButtons.slice(1),
                        ];
                        this.$ui.configure("overflowMenuButtons", newOverflowMenuButtons);
                    }

                    if (qualityConds) {
                        var leastDiff = Number.MAX_VALUE;
                        var bestStream = null;

                        var bestAudio = 0;

                        const tracks = player
                            .getVariantTracks()
                            .filter(track => track.language == lang || track.language == "und");

                        // Choose the best audio stream
                        if (quality >= 480)
                            tracks.forEach(track => {
                                const audioBandwidth = track.audioBandwidth;
                                if (audioBandwidth > bestAudio) bestAudio = audioBandwidth;
                            });

                        // Find best matching stream based on resolution and bitrate
                        tracks
                            .sort((a, b) => a.bandwidth - b.bandwidth)
                            .forEach(stream => {
                                if (stream.audioBandwidth < bestAudio) return;

                                const diff = Math.abs(quality - stream.height);
                                if (diff < leastDiff) {
                                    leastDiff = diff;
                                    bestStream = stream;
                                }
                            });

                        player.selectVariantTrack(bestStream);
                    }

                    this.video.subtitles.map(subtitle => {
                        player.addTextTrackAsync(
                            subtitle.url,
                            subtitle.code,
                            "subtitles",
                            subtitle.mimeType,
                            null,
                            subtitle.name,
                        );
                    });
                    videoEl.volume = this.getPreferenceNumber("volume", 1);
                    const rate = this.getPreferenceNumber("rate", 1);
                    videoEl.playbackRate = rate;
                    videoEl.defaultPlaybackRate = rate;

                    const autoDisplayCaptions = this.getPreferenceBoolean("autoDisplayCaptions", false);
                    this.$player.setTextTrackVisibility(autoDisplayCaptions);

                    const prefSubtitles = this.getPreferenceString("subtitles", "");
                    if (prefSubtitles !== "") {
                        const textTracks = this.$player.getTextTracks();
                        const subtitleIdx = textTracks.findIndex(textTrack => textTrack.language == prefSubtitles);
                        if (subtitleIdx != -1) {
                            this.$player.setTextTrackVisibility(true);
                            this.$player.selectTextTrack(textTracks[subtitleIdx]);
                        }
                    }
                })
                .catch(e => {
                    console.error(e);
                    this.error = e.code;
                });

            // expand the player to fullscreen when the fullscreen query equals true
            if (this.$route.query.fullscreen === "true" && !this.$ui.getControls().isFullScreenEnabled())
                this.$ui.getControls().toggleFullScreen();
        },
        async updateProgressDatabase(time) {
            // debounce
            if (new Date().getTime() - this.lastUpdate < 500) return;
            this.lastUpdate = new Date().getTime();

            if (!this.initialSeekComplete || !this.video.id || !window.db) return;

            var tx = window.db.transaction("watch_history", "readwrite");
            var store = tx.objectStore("watch_history");
            var request = store.get(this.video.id);
            request.onsuccess = function (event) {
                var video = event.target.result;
                if (video) {
                    video.currentTime = time;
                    store.put(video);
                }
            };
        },
        seek(time) {
            if (this.$refs.videoEl) {
                this.$refs.videoEl.currentTime = time;
            }
        },
        adjustPlaybackSpeed(newSpeed) {
            const normalizedSpeed = Math.min(4, Math.max(0.25, newSpeed));
            this.$player.trickPlay(normalizedSpeed);
        },
        setSpeedFromInput() {
            try {
                const newSpeed = Number(this.playbackSpeedInput);
                this.adjustPlaybackSpeed(newSpeed);
            } catch (err) {
                alert(this.$t("actions.invalid_input"));
            }
            this.showSpeedModal = false;
        },
        updateMarkers() {
            const markers = this.$refs.container.querySelector(".shaka-ad-markers");
            const array = ["to right"];
            this.sponsors?.segments?.forEach(segment => {
                const start = (segment.segment[0] / this.video.duration) * 100;
                const end = (segment.segment[1] / this.video.duration) * 100;

                var color = [
                    "sponsor",
                    "selfpromo",
                    "interaction",
                    "poi_highlight",
                    "intro",
                    "outro",
                    "preview",
                    "filler",
                    "music_offtopic",
                ].includes(segment.category)
                    ? `var(--spon-seg-${segment.category})`
                    : "var(--spon-seg-default)";

                array.push(`transparent ${start}%`);
                array.push(`${color} ${start}%`);
                array.push(`${color} ${end}%`);
                array.push(`transparent ${end}%`);
            });

            if (array.length <= 1) {
                return;
            }

            if (markers) markers.style.background = `linear-gradient(${array.join(",")})`;
        },
        updateSponsors() {
            if (this.getPreferenceBoolean("showMarkers", true)) {
                this.shakaPromise.then(() => {
                    this.updateMarkers();
                });
            }
        },
        setupSeekbarPreview() {
            if (!this.video.previewFrames) return;
            let seekBar = document.querySelector(".shaka-seek-bar");
            // load the thumbnail preview when the user moves over the seekbar
            seekBar.addEventListener("mousemove", e => {
                this.isHoveringTimebar = true;
                const position = (e.offsetX / e.target.offsetWidth) * this.video.duration;
                this.showSeekbarPreview(position * 1000);
            });
            // hide the preview when the user stops hovering the seekbar
            seekBar.addEventListener("mouseout", () => {
                this.isHoveringTimebar = false;
                this.$refs.previewContainer.style.display = "none";
            });
        },
        async showSeekbarPreview(position) {
            const frame = this.getFrame(position);
            const originalImage = await this.loadImage(frame.url);
            if (!this.isHoveringTimebar) return;

            const seekBar = document.querySelector(".shaka-seek-bar");
            const container = this.$refs.previewContainer;
            const canvas = this.$refs.preview;
            const ctx = canvas.getContext("2d");

            const offsetX = frame.positionX * frame.frameWidth;
            const offsetY = frame.positionY * frame.frameHeight;

            canvas.width = frame.frameWidth > 100 ? frame.frameWidth : frame.frameWidth * 2;
            canvas.height = frame.frameWidth > 100 ? frame.frameHeight : frame.frameHeight * 2;
            // draw the thumbnail preview into the canvas by cropping only the relevant part
            ctx.drawImage(
                originalImage,
                offsetX,
                offsetY,
                frame.frameWidth,
                frame.frameHeight,
                0,
                0,
                canvas.width,
                canvas.height,
            );

            // calculate the thumbnail preview offset and display it
            const centerOffset = position / this.video.duration / 10;
            const left = centerOffset - ((0.5 * canvas.width) / seekBar.clientWidth) * 100;
            const maxLeft =
                ((seekBar.clientWidth - canvas.clientWidth) / seekBar.clientWidth) * 100 - this.seekbarPadding;

            this.currentTime = position / 1000;

            container.style.left = `max(${this.seekbarPadding}%, min(${left}%, ${maxLeft}%))`;
            container.style.display = "flex";
        },
        // ineffective algorithm to find the thumbnail corresponding to the currently hovered position in the video
        getFrame(position) {
            let startPosition = 0;
            const framePage = this.video.previewFrames.at(-1);
            for (let i = 0; i < framePage.urls.length; i++) {
                for (let positionY = 0; positionY < framePage.framesPerPageY; positionY++) {
                    for (let positionX = 0; positionX < framePage.framesPerPageX; positionX++) {
                        const endPosition = startPosition + framePage.durationPerFrame;
                        if (position >= startPosition && position <= endPosition) {
                            return {
                                url: framePage.urls[i],
                                positionX: positionX,
                                positionY: positionY,
                                frameWidth: framePage.frameWidth,
                                frameHeight: framePage.frameHeight,
                            };
                        }
                        startPosition = endPosition;
                    }
                }
            }
            return null;
        },
        // creates a new image from an URL
        loadImage(url) {
            return new Promise(r => {
                const i = new Image();
                i.onload = () => r(i);
                i.src = url;
            });
        },
        destroy(hotkeys) {
            if (this.$ui && !document.pictureInPictureElement) {
                this.$ui.destroy();
                this.$ui = undefined;
                this.$player = undefined;
            }
            if (this.$player) {
                this.$player.destroy();
                if (!document.pictureInPictureElement) this.$player = undefined;
            }
            if (hotkeys) this.$hotkeys?.unbind();
            this.$refs.container?.querySelectorAll("div").forEach(node => node.remove());
        },
    },
};
</script>

<style>
:root {
    --player-base: rgba(255, 255, 255, 0.3);
    --player-buffered: rgba(255, 255, 255, 0.54);
    --player-played: rgba(255, 0, 0);

    --spon-seg-sponsor: #00d400;
    --spon-seg-selfpromo: #ffff00;
    --spon-seg-interaction: #cc00ff;
    --spon-seg-poi_highlight: #ff1684;
    --spon-seg-intro: #00ffff;
    --spon-seg-outro: #0202ed;
    --spon-seg-preview: #008fd6;
    --spon-seg-filler: #7300ff;
    --spon-seg-music_offtopic: #ff9900;
    --spon-seg-default: white;
}

.player-container {
    @apply max-h-75vh min-h-64;
    background: #000;
}
[efy_theme="dark_black"] .player-container {
    box-shadow: 0 0 0 1.5rem var(--efy_bg1);
}
.shaka-video-container:-webkit-full-screen {
    max-height: none !important;
}

/*Captions*/
.shaka-text-wrapper * {
    text-align: left !important;
    font-family: "nunito" !important;
}
.shaka-text-wrapper > span {
    background: #0008 !important;
    backdrop-filter: blur(20rem);
    border-radius: var(--efy_radius);
    padding: 6rem 10rem;
    line-height: 24rem;
}
.shaka-text-wrapper > span:empty {
    display: none !important;
}
.shaka-text-wrapper > span * {
    background: transparent !important;
}
/* apply to all spans that don't include multiple other spans to avoid the style being applied to the text container too when the subtitles are two lines */
.shaka-text-wrapper > span > span *:first-child:last-child {
    background: transparent !important;
    color: #fff;
    padding: 0;
    font-size: 22rem !important;
}
@media (max-width: 639px) {
    .shaka-text-wrapper > span > span *:first-child:last-child {
        font-size: 16rem !important;
    }
}

/*Shaka Player*/
html .shaka-range-element,
html .shaka-range-element:focus {
    border: none !important;
    box-shadow: none !important;
    outline: none !important;
}
.material-icons-round {
    font-family: "Material Icons Round" !important;
}
.shaka-bottom-controls .material-icons-round,
.shaka-current-time {
    -webkit-text-fill-color: #fff !important;
    filter: none !important;
    opacity: 1 !important;
    box-shadow: none !important;
}
.shaka-bottom-controls .material-icons-round {
    box-shadow: none !important;
}
.shaka-overflow-menu,
.shaka-settings-menu {
    background: var(--efy_text2) !important;
    border-radius: var(--efy_radius) !important;
    padding: 16rem !important;
    height: fit-content !important;
    max-height: 256rem !important;
    box-shadow: inset 0 0 0 1.5px var(--efy_bg1) !important;
}
.shaka-overflow-menu label,
.shaka-settings-menu label,
.shaka-overflow-menu button,
.shaka-settings-menu button {
    -webkit-text-fill-color: var(--efy_text) !important;
    margin: 1.6rem 0 !important;
    place-content: start;
    width: 100%;
    .shaka-overflow-menu-only {
        width: fit-content;
    }
}
.shaka-overflow-menu .material-icons-round,
.shaka-settings-menu .material-icons-round {
    -webkit-text-fill-color: var(--efy_text) !important;
}

.shaka-overflow-menu button:hover,
.shaka-settings-menu button:hover {
    background: var(--efy_bg1) !important;
    box-shadow: inset 0 0 0 1.5px var(--efy_bg1);
}
.shaka-controls-container {
    border-radius: var(--efy_radius) !important;
}

.skip-segment-button {
    z-index: 1000;
    position: absolute;
    transform: translate(0, -50%);
    top: 50%;
    right: 0;
    border-right: 0;
    border-radius: var(--efy_radius) 0 0 var(--efy_radius);
    display: flex;
    align-items: center;
    justify-content: center;
}
</style>
