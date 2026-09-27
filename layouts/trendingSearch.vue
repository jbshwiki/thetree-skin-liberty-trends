<!--
  실시간 검색어 사이드바 카드 (Liberty 스킨용)
  원본: pyhokwiki의 수정된 Liberty 빌드 결과물(index-*.js / index-*.css)에서 복원.

  동작
  - /Search?q=... (전체 검색) 페이지에 들어오면 검색어를 POST로 보고
  - 30초마다 순위를 새로 받아오고
  - 평소엔 1위부터 3초마다 한 줄씩 돌아가며 보여주다가, 마우스를 올리면 10개 전체 펼침
-->
<template>
    <div class="trending-search-space">
        <section
            class="trending-search-card"
            :class="{ expanded }"
            tabindex="0"
            @mouseenter="openList"
            @mouseleave="closeList"
            @focusin="openList"
            @focusout="onFocusOut"
        >
            <div class="trending-search-header">
                <span class="fa fa-bar-chart" aria-hidden="true"></span>
                <strong>실시간 검색어</strong>
            </div>

            <ol v-if="visibleItems.length" class="trending-search-list">
                <li v-for="item in visibleItems" :key="item.rank + '-' + item.query" class="trending-search-item">
                    <span class="trending-search-rank" :class="{ top: item.rank <= 3 }">{{ item.rank }}</span>
                    <nuxt-link :to="searchLink(item.query)" class="trending-search-link" :title="item.query">
                        {{ item.query }}
                    </nuxt-link>
                </li>
            </ol>
            <div v-else class="trending-search-empty">
                {{ loaded ? '검색 기록이 없습니다.' : '불러오는 중...' }}
            </div>
        </section>
    </div>
</template>

<script>
const API = '/api/wiki-search-trends';

export default {
    data() {
        return {
            items: [],
            loaded: false,
            currentIndex: 0,
            expanded: false,
            cycleTimer: null,
            refreshTimer: null,
            lastRecordedRoute: ''
        };
    },
    computed: {
        visibleItems() {
            if (this.expanded) return this.items;
            const item = this.items[this.currentIndex];
            return item ? [item] : [];
        }
    },
    watch: {
        '$route.fullPath'() {
            this.recordSearchFromRoute();
        }
    },
    mounted() {
        window.addEventListener('wiki-search-trends-updated', this.fetchTrends);
        this.recordSearchFromRoute();
        this.fetchTrends();

        this.cycleTimer = setInterval(() => {
            if (this.expanded || this.items.length <= 1) return;
            this.currentIndex = (this.currentIndex + 1) % this.items.length;
        }, 3000);

        this.refreshTimer = setInterval(() => this.fetchTrends(), 30000);
    },
    // 원본은 beforeDestroy(Vue 2 이름)라 Vue 3에서 호출되지 않았음 → beforeUnmount로 수정
    beforeUnmount() {
        window.removeEventListener('wiki-search-trends-updated', this.fetchTrends);
        if (this.cycleTimer) clearInterval(this.cycleTimer);
        if (this.refreshTimer) clearInterval(this.refreshTimer);
    },
    methods: {
        openList() {
            this.expanded = true;
        },
        closeList() {
            this.expanded = false;
        },
        onFocusOut(e) {
            if (!e.relatedTarget || !this.$el.contains(e.relatedTarget)) this.closeList();
        },
        searchLink(query) {
            return '/w/' + encodeURIComponent(query);
        },
        async fetchTrends() {
            try {
                const res = await fetch(API, {
                    method: 'GET',
                    cache: 'no-store',
                    headers: { Accept: 'application/json' }
                });
                if (!res.ok) return;
                const data = await res.json();
                this.items = Array.isArray(data.items) ? data.items.slice(0, 10) : [];
                if (this.currentIndex >= this.items.length) this.currentIndex = 0;
            } catch (e) {
                console.warn('[search-trends] 조회 실패', e);
            } finally {
                this.loaded = true;
            }
        },
        // 전체 검색 페이지(/Search?q=...)에 들어왔을 때만 기록.
        // 존재하는 문서인지 판정은 서버가 한다.
        recordSearchFromRoute() {
            if (String(this.$route.path || '').toLowerCase() !== '/search') return;

            const raw = Array.isArray(this.$route.query.q) ? this.$route.query.q[0] : this.$route.query.q;
            const query = String(raw || '').replace(/\s+/g, ' ').trim();
            if (!query) return;

            const key = this.$route.fullPath + '\0' + query;
            if (key === this.lastRecordedRoute) return;
            this.lastRecordedRoute = key;

            this.recordQuery(query);
        },
        async recordQuery(query) {
            try {
                await fetch(API, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json', Accept: 'application/json' },
                    body: JSON.stringify({ query }),
                    keepalive: true
                });
                await this.fetchTrends();
            } catch (e) {
                console.warn('[search-trends] 기록 실패', e);
            }
        }
    }
};
</script>

<style scoped>
/* 카드는 absolute로 떠 있어서, 펼쳐져도 아래 '최근 변경' 박스를 밀지 않고 위에 덮는다 */
.trending-search-space {
    position: relative;
    height: 5.65rem;
    margin-bottom: .75rem;
}

.trending-search-card {
    position: absolute;
    top: 0;
    right: 0;
    left: 0;
    z-index: 30;
    overflow: hidden;
    min-height: 5.65rem;
    border: 1px solid rgba(127, 127, 127, .32);
    border-radius: .25rem;
    background: var(--article-background-color, #fff);
    color: var(--text-color, #373a3c);
    box-shadow: 0 1px 3px #00000014;
    outline: none;
}

.trending-search-card.expanded {
    overflow: visible;
    box-shadow: 0 6px 18px #00000038;
}

.trending-search-header {
    display: flex;
    align-items: center;
    gap: .65rem;
    height: 3rem;
    padding: 0 1rem;
    border-bottom: 1px solid rgba(127, 127, 127, .16);
    font-size: 1rem;
}

.trending-search-header .fa {
    color: #888;
}

.trending-search-list {
    margin: 0;
    padding: .3rem .75rem .6rem;
    list-style: none;
    background: inherit;
}

.trending-search-item {
    display: flex;
    align-items: center;
    min-width: 0;
    height: 2rem;
    border-bottom: 1px solid rgba(127, 127, 127, .08);
    line-height: 2rem;
}

.trending-search-item:last-child {
    border-bottom: 0;
}

.trending-search-rank {
    flex: 0 0 1.75rem;
    margin-right: .3rem;
    color: var(--text-color, #373a3c);
    font-weight: 600;
    text-align: center;
}

.trending-search-rank.top {
    color: var(--liberty-brand-color, #4188f1);
}

.trending-search-link {
    overflow: hidden;
    min-width: 0;
    color: var(--liberty-brand-color, #4188f1);
    text-decoration: none;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.trending-search-link:hover,
.trending-search-link:focus {
    text-decoration: underline;
}

.trending-search-empty {
    padding: .65rem 1rem .9rem;
    color: #888;
    font-size: .9rem;
}
</style>

<style>
/* 다크 모드 (scoped 밖에서 body 클래스를 봐야 해서 전역 스타일) */
body.theseed-dark-mode .Liberty .trending-search-card {
    background-color: #000 !important;
    border-color: #666 !important;
    color: #ddd !important;
}

body.theseed-dark-mode .Liberty .trending-search-header {
    background-color: #16171a !important;
    border-bottom-color: #444 !important;
    color: #e8cf91 !important;
}

body.theseed-dark-mode .Liberty .trending-search-card ol {
    background-color: #000 !important;
}

body.theseed-dark-mode .Liberty .trending-search-card li {
    background-color: #000 !important;
    border-color: #2d2f34 !important;
    color: #ddd !important;
}

body.theseed-dark-mode .Liberty .trending-search-card li a,
body.theseed-dark-mode .Liberty .trending-search-card li a:visited {
    color: #ddd !important;
}

body.theseed-dark-mode .Liberty .trending-search-card li:hover,
body.theseed-dark-mode .Liberty .trending-search-card li:hover * {
    background-color: #16171a !important;
    color: #fff !important;
}

body.theseed-dark-mode .Liberty .trending-search-rank {
    color: #fff !important;
    font-weight: 700;
}

body.theseed-dark-mode .Liberty .trending-search-empty {
    background-color: #000 !important;
    color: #aaa !important;
}
</style>
