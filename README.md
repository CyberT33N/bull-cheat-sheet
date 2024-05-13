# bull-cheat-sheet



# Example

```javascript
before(async () => {
    const queue = new Queue('thumbnail-processing', {
        redis: {
            host: process.env.REDIS_HOST,
            port: process.env.REDIS_PORT,
            password: process.env.REDIS_PASSWORD
        }
    })

    // Clear all queues
    await queue.obliterate({ force: true })

    const redisWrapper = await cache({
        host: process.env.REDIS_HOST,
        port: process.env.REDIS_PORT,
        password: process.env.REDIS_PASSWORD || undefined
    }, 'redis')

    redisClient = redisWrapper.getClient()
})

describe('save', () => {
    it.only('should create two create mailer template and add thumbnail generation to que', async () => {
        const thumbnailName = "test"
        const dataWithNewThumbnail = { ...data, "thumbnail": thumbnailName }

        const res = await Model.create(dataWithNewThumbnail)
        expect(res.thumbnail).to.be.equal(thumbnailName)

        await new Promise(resolve => setTimeout(resolve, 2000))

        const doc = await redisClient.HGETALL('bull:thumbnail-processing:1')
        const parsedData = JSON.parse(doc.data)
        expect(parsedData.modelName).to.be.equal(modelName)
        expect(parsedData.projectId).to.be.equal(projectId)
        expect(parsedData.doc.thumbnail).to.be.equal(thumbnailName)
    })
})
```

```javascript
const Queue = require('bull')

const queue = new Queue('thumbnail-processing', {
    redis: {
        host: process.env.REDIS_HOST,
        port: process.env.REDIS_PORT,
        password: process.env.REDIS_PASSWORD
    }
})

/**
 * Process the template thumbnail.
 *
 * @param {Object} job - The job object containing the data.
 * @param {string} job.projectId - The ID of the project.
 * @param {Object} job.doc - The document object.
 * @param {string} job.modelName - The name of the model.
 * @throws {BaseError} If there is an error processing the thumbnail.
 */
const _processTemplateThumbnail = async job => {
    try {
        const { projectId, doc, modelName } = job.data
        await processTemplateThumbnail(projectId, doc, modelName)
    } catch (e) {
        throw new BaseError('thumbnail-processing - Error processing thumbnail', e)
    }
}

queue.process('thumbnail', _processTemplateThumbnail)

queue.on('completed', job => {
    console.log(`[ THUMBNAIL QUEUE] - Job ${job.id} abgeschlossen`)
})

queue.on('failed', (job, e) => {
    console.error(`[THUMBNAIL QUEUE] - Job ${job.id} fehlgeschlagen:`, e)
})

/**
 *
 * @param schema
 * @param options
 * @constructor
 */
const Templates = (schema, options) => {
    /**
     * insertMany will be maybe used in future
     */
    schema.post(['save'], function(doc, next) {
        const projectId = getProjectIdFromDoc(this)

        /*
            We do not use await here on purpose. We do not want that user have to wait
            In order that the app does not crash on uncaughtException we catch the error and log it.
        */
        queue.add('thumbnail', { projectId, doc, modelName })
            .catch((e) => {
                console.error("Plugin - findOneAndUpdate - Can not process template thumbnail. Error: ", e)
            })

        next()
    })

}
```


# clear ques
```javascript
await queue.obliterate({ force: true })
```
