throttle(['one-per-node-throttle']) {
    node('linux') {
        pipeline = load 'JenkinsfileScript.groovy'
        pipeline.call()
    }
}
