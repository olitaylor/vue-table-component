<template>
    <nav v-if="shouldShowPagination">
        <ul class="pagination justify-content-center">
            <li class="page-item" :class="{ active: isActive(page) }" v-for="page in pages">
                <a class="page-link" @click="pageClicked(page)">{{ page }}</a>
            </li>
        </ul>
    </nav>
</template>

<script>
    import range from 'lodash/range';

    export default {
        props: {
            pagination: {
                type: Object,
                default: () => ({}),
            },
        },

        computed: {
            pages() {
                return this.last_page === undefined
                    ? []
                    : range(1, this.last_page + 1);
            },

            shouldShowPagination() {
                if (this.last_page === undefined) {
                    return false;
                }

                if (this.pagination.count === 0) {
                    return false;
                }

                return this.last_page > 1;
            },
        },

        methods: {
            isActive(page) {
                const currentPage = this.current_page || 1;

                return currentPage === page;
            },

            pageClicked(page) {
                if (this.current_page === page) {
                    return;
                }

                this.$emit('pageChange', page);
            },
        },
    };
</script>
