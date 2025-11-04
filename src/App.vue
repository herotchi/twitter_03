<template>
    <nav
        class="navbar navbar-expand-lg bg-body-tertiary bg-dark border-bottom border-bottom-dark mb-3"
        data-bs-theme="dark"
    >
        <div class="container-fluid">
            <a class="navbar-brand" href="#">王様の耳はロバの耳</a>
            <!--<button
                class="navbar-toggler"
                type="button"
                data-bs-toggle="collapse"
                data-bs-target="#navbarSupportedContent"
                aria-controls="navbarSupportedContent"
                aria-expanded="false"
                aria-label="Toggle navigation"
            >
                <span class="navbar-toggler-icon"></span>
            </button>-->
        </div>
    </nav>
    <div class="container mt-5">
        <div class="row justify-content-center">
            <div class="col-lg-6">
                <div class="card shadow-sm p-3 mb-4">
                    <div class="card-body">
                        <form @submit.prevent="addTweet">
                            <div class="mb-3">
                                <textarea
                                    v-model="newTweet"
                                    placeholder="いまどうしてる？"
                                    rows="3"
                                    class="form-control"
                                ></textarea>
                            </div>
                            <!-- 文字数カウント表示 -->
                            <p
                                :class="[
                                    'text-end',
                                    isOverLimit ? 'text-danger' : 'text-muted',
                                ]"
                            >
                                {{ newTweet.length }}/{{ MAX_LENGTH }}
                            </p>
                            <div class="">
                                <button
                                    type="submit"
                                    class="btn btn-primary w-100"
                                    :disabled="!newTweet.trim()"
                                >
                                    シャウト
                                </button>
                            </div>
                            <!-- エラーメッセージ -->
                            <div
                                v-if="errorMessage"
                                class="alert alert-danger mt-2 mb-0"
                            >
                                {{ errorMessage }}
                            </div>
                        </form>
                    </div>
                </div>

                <transition-group name="fade" tag="div" class="tweet-list">
                    <div
                        v-for="tweet in displayTweets"
                        :key="tweet.id"
                        class="card mb-3 shadow-sm"
                    >
                        <div class="card-body">
                            <p class="card-text mb-2 tweet-content">
                                {{ tweet.content }}
                            </p>
                            <p class="card-subtitle text-muted small mb-0">
                                {{ tweet.createdAt }}
                            </p>
                            <button
                                class="btn btn-sm btn-outline-danger position-absolute bottom-0 end-0 m-2"
                                @click="deleteTweet(tweet.id)"
                            >
                                削除
                            </button>
                        </div>
                    </div>
                </transition-group>
                <!-- ▼ ローディング中スピナー -->
                <div v-if="loading" class="text-center my-3">
                    <div class="spinner-border text-primary" role="status">
                        <span class="visually-hidden">Loading...</span>
                    </div>
                </div>

                <!-- ▼ 全件表示時メッセージ -->
                <!--<div
                    v-if="!loading && displayTweets.length >= allTweets.length"
                    class="text-center text-muted mt-3"
                >
                    すべてのツイートを表示しました
                </div>-->
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, watch, onMounted, onBeforeUnmount, computed } from "vue";

interface Tweet {
    id: string;
    content: string;
    createdAt: string;
}

const STORAGE_KEY = "shout";
// 1回で読み込む件数
const LOAD_COUNT = 5;

const MAX_LENGTH = 140;

const allTweets = ref<Tweet[]>([]);
const displayTweets = ref<Tweet[]>([]);
const newTweet = ref("");
// 文字数超過チェック
const isOverLimit = computed(() => newTweet.value.length > MAX_LENGTH);
const errorMessage = ref("");

// ローディング状態
const loading = ref(false);

// 起動時に保存データを読み込む
onMounted(() => {
    const saved = localStorage.getItem(STORAGE_KEY);
    if (saved) {
        allTweets.value = JSON.parse(saved);
        displayTweets.value = allTweets.value.slice(0, LOAD_COUNT);
    }
    window.addEventListener("scroll", handleScroll);
});

// コンポーネントが破棄される直前に呼ばれる
// 登録していたスクロールイベントを解除して、不要な処理やメモリリークを防ぐ
onBeforeUnmount(() => {
    window.removeEventListener("scroll", handleScroll);
});

// 変更があれば保存
watch(
    allTweets,
    (val) => localStorage.setItem(STORAGE_KEY, JSON.stringify(val)),
    { deep: true }
);

// 一意のIDを生成する関数
const generateId = () =>
    Date.now().toString() + Math.random().toString(36).substring(2, 8);

// ツイート追加
const addTweet = () => {
    console.log(isOverLimit.value);
    errorMessage.value = "";
    // 未入力チェック
    if (!newTweet.value.trim()) return;
    // 文字数チェック
    if (isOverLimit.value) {
        errorMessage.value = `シャウトは${MAX_LENGTH}文字以内で入力してください。`;
        return;
    }

    const newItem: Tweet = {
        id: generateId(),
        content: newTweet.value,
        createdAt: new Date().toLocaleString(),
    };

    allTweets.value.unshift(newItem);
    displayTweets.value.unshift(newItem);
    newTweet.value = "";
};

// ツイート削除
const deleteTweet = (id: string) => {
    if (!confirm("このシャウトを削除しますか？")) return;
    allTweets.value = allTweets.value.filter((t) => t.id !== id);
    displayTweets.value = displayTweets.value.filter((t) => t.id !== id);
};

// スクロールイベント
const handleScroll = async () => {
    // すでにローディング中なら無視
    if (loading.value) return;

    const scrollPosition = window.innerHeight + window.scrollY;
    const pageHeight = document.body.offsetHeight;

    if (scrollPosition >= pageHeight - 50) {
        await loadMore();
    }
};

const hasMore = computed(
    () => displayTweets.value.length < allTweets.value.length
);

// 次の5件を読み込み
const loadMore = async () => {
    // 追加読み込み不要 or 読み込み中なら抜ける
    if (!hasMore.value || loading.value) return;

    loading.value = true;
    try {
        // 実際のAPI呼び出しがある場合はここで await fetch/axios...
        await new Promise((resolve) => setTimeout(resolve, 1000));

        const start = displayTweets.value.length;
        const next = allTweets.value.slice(start, start + LOAD_COUNT);
        if (next.length) {
            displayTweets.value.push(...next);
        }
    } catch (err) {
        console.error("loadMore error:", err);
        // 必要ならユーザーに通知（トースト等）を行う
    } finally {
        loading.value = false; // 必ず実行してロックを解除
    }
};
</script>

<style scoped>
/* フェードアニメーションのCSS */
.fade-enter-active,
.fade-leave-active {
    transition: all 0.4s ease;
}
.fade-enter-from,
.fade-leave-to {
    opacity: 0;
    transform: translateY(10px);
}
.tweet-content {
    /* 改行や空白をそのまま表示 */
    white-space: pre-wrap;
    /* 長い単語で折り返す */
    word-break: break-word;
}
</style>
