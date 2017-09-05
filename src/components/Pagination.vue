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
                return this.pagination.last_page === undefined
                    ? []
                    : range(1, this.pagination.last_page + 1);
            },

            shouldShowPagination() {
                if (this.pagination.last_page === undefined) {
                    return false;
                }

                return this.pagination.last_page > 1;
            },
        },

        methods: {
            isActive(page) {
                const currentPage = this.pagination.current_page || 1;

                return currentPage === page;
            },

            pageClicked(page) {
                if (this.pagination.current_page === page) {
                    return;
                }

                this.$emit('pageChange', page);
            },
        },
    };
</script>
