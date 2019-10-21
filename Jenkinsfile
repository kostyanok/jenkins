throttle(['one-per-node-throttle']) {
    node('linux') {
        checkout scm
        pipeline = load 'JenkinsfileScript'
        pipeline.call()
    }
}
