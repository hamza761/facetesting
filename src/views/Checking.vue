<template>
<div>
    {{ time }}
    <div>{{count}}
        <button @click="setCInterval">
        start timer
        </button>
    </div>
</div>
</template>

<script>
let callbackMethod = () => console.log("in itial")
export default {
    data() {
        return {
            time: null,
            count: 0
        }
    },
    methods: {
        setCInterval () {
            console.time('checking'+this.count)
            callbackMethod = () => console.log("in interval")
            const interval = setInterval(() => {
                console.timeEnd('checking'+this.count)
                this.count++
                if (this.count > 2) {
                    callbackMethod = () => console.log("completed")
                    clearInterval(interval)
                }
                console.time('checking'+this.count)
            }, 1000);
        },
        setDate () {
            setInterval(() => {
                // console.log('in interval')
                callbackMethod()
                this.time = new Date()
            }, 1000);
        }
    },
    mounted() {
        this.setDate()
    },
}
</script>