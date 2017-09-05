<template>
    <nav v-if="shouldShowPagination">
        <ul class="pagination pagination-flat pagination-sm twbs-visible-pages" v-if="itemsCount > 0">
            <li :class="{ disabled: pagination.totalPages === 1 }"><a href="#" v-on:click.prevent="pageClicked(pagination.currentPage - 1)">&lsaquo;</a></li>
            <li v-for="p in numbers" :class="{ active: p === pagination.currentPage }"><a href="#" @click="pageClicked(p)">{{ p }}</a></li>
            <li :class="{ disabled: pagination.currentPage === pagination.totalPages }"><a href="#" v-on:click.prevent="pageClicked(pagination.currentPage + 1)">&rsaquo;</a></li>
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
            numbers() {
                let numbers = [];
                let from = this.pagination.currentPage - 2;
                let to = this.pagination.currentPage + 2
                if(from < 1) {
                    from = 1;
                }
                if(to > this.pagination.totalPages) {
                    to = this.pagination.totalPages;
                }
                let m = -(to - from - 4);
                if(m > 0) {
                    if(from > 1) {
                        from = from - m > 0 ? from - m : 1;
                    } else if(to < this.pagination.totalPages) {
                        to = to + m > this.pagination.totalPages ? this.pagination.totalPages : to + m;
                    }
                }
                for(let tmp = from; tmp <= to; tmp++) {
                    numbers.push(tmp);
                }
                return numbers;
            }

            shouldShowPagination() {
                if (this.pagination.totalPages === undefined) {
                    return false;
                }

                if (this.pagination.count === 0) {
                    return false;
                }

                return this.pagination.totalPages > 1;
            },
        },

        methods: {
            pageClicked(page) {
                if(page > 0 && page <= this.pagination.totalPages) {
                    this.$emit('pageChange', page)
                }
            },
        },
    };
</script>
