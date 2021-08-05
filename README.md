# mongodb
* Collections
* Documents (JSON Data)
* mongoose v5.13.5
* Mongoose MIDLEWAREs ALSO Know as HOOKs

### Connections:
```env
MONGO_URI = 'mongodb://localhost:27017/dbname' // if it does not exists it will create the db first
```
```js
const moongoose = require('mongoose');

const connectDB = async () => {
  const conn = await mongoose.connect(Config.MONGO_URI, {
    useNewUrlParser: true, // if we dont use this it will print follwoing warning "DeprecationWarning: current URL string parser is deprecated,
    // and will be removed in a future version. To use the new parser, pass option { useNewUrlParser: true } to MongoClient.connect."
    // Optional. Use this if you create a lot of connections and don't want to copy/paste `{ useNewUrlParser: true }` "mongoose.set('useNewUrlParser', true);"
    useCreateIndex: true,
    // If you define indexes in your Mongoose schemas, you'll see the below deprecation warning.
    // DeprecationWarning: collection.ensureIndex is deprecated. Use createIndexes instead.
    // By default, Mongoose 5.x calls the MongoDB driver's ensureIndex() function. The MongoDB driver deprecated this function in favor of createIndex().
    // Set the useCreateIndex global option to opt in to making Mongoose use createIndex() instead.
    useFindAndModify: false,
    // If you use Model.findOneAndUpdate(), by default you'll see one of the below deprecation warnings.
    // DeprecationWarning: Mongoose: `findOneAndUpdate()` and `findOneAndDelete()` without the `useFindAndModify` option set to false are deprecated.
    // See: https://mongoosejs.com/docs/deprecations.html#findandmodify
    // DeprecationWarning: collection.findAndModify is deprecated. Use findOneAndUpdate, findOneAndReplace or findOneAndDelete instead.
    // to set it globally mongoose.set('useFindAndModify', false);
    useUnifiedTopology: true
  });
  
  console.log(`Connected: ${conn.connection.host}`);
}

// NOW CALL THIS METHOD in SERVER.JS so that mongoose will have connection when we create/update data
```

### Read about mongodb depreciations:
https://mongoosejs.com/docs/deprecations.html

### _id is by default created and with primary index

### Create Model:
```js
const mongoose = require('mongoose');
const slugify = require('slugify');
const geocoder = require('../utils/geocoder');

const BootcampSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: [true, 'Please add a name'],
      unique: true,
      trim: true,
      maxlength: [50, 'Name can not be more than 50 characters']
    },
    slug: String,
    description: {
      type: String,
      required: [true, 'Please add a description'],
      maxlength: [500, 'Description can not be more than 500 characters']
    },
    website: {
      type: String,
      match: [
        /https?:\/\/(www\.)?[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b([-a-zA-Z0-9()@:%_\+.~#?&//=]*)/,
        'Please use a valid URL with HTTP or HTTPS'
      ]
    },
    phone: {
      type: String,
      maxlength: [20, 'Phone number can not be longer than 20 characters']
    },
    email: {
      type: String,
      match: [
        /^\w+([\.-]?\w+)*@\w+([\.-]?\w+)*(\.\w{2,3})+$/,
        'Please add a valid email'
      ]
    },
    address: {
      type: String,
      required: [true, 'Please add an address']
    },
    location: {
      // GeoJSON Point
      type: {
        type: String,
        enum: ['Point']   // can only be Point
      },
      coordinates: {
        type: [Number], // array of numbers
        index: '2dsphere' // to add index on cordinates   https://docs.mongodb.com/manual/core/2dsphere/
      },
      formattedAddress: String,
      street: String,
      city: String,
      state: String,
      zipcode: String,
      country: String
    },
    careers: {
      // Array of strings
      type: [String],
      required: true,
      enum: [   // these are the only avilable values that this filed can have
        'Web Development',
        'Mobile Development',
        'UI/UX',
        'Data Science',
        'Business',
        'Other'
      ]
    },
    averageRating: {
      type: Number,
      min: [1, 'Rating must be at least 1'],
      max: [10, 'Rating must can not be more than 10']
    },
    averageCost: Number,
    photo: {
      type: String,
      default: 'no-photo.jpg' // Default Value
    },
    housing: {
      type: Boolean,
      default: false
    },
    jobAssistance: {
      type: Boolean,
      default: false
    },
    jobGuarantee: {
      type: Boolean,
      default: false
    },
    acceptGi: {
      type: Boolean,
      default: false
    },
    createdAt: {
      type: Date,
      default: Date.now
    },
    user: {
      type: mongoose.Schema.ObjectId,
      ref: 'User',
      required: true
    }
  },
  {
    toJSON: { virtuals: true },
    toObject: { virtuals: true }
  }
);

// Mongoose MIDLEWAREs ALSO Know as HOOKs
// We can have pre(before operation) and post(after operation) HOOKS

// Create bootcamp slug from the name
BootcampSchema.pre('save', function(next) {
  this.slug = slugify(this.name, { lower: true });
  next();
});

// Geocode & create location field
BootcampSchema.pre('save', async function(next) {
  const loc = await geocoder.geocode(this.address);
  this.location = {
    type: 'Point',
    coordinates: [loc[0].longitude, loc[0].latitude],
    formattedAddress: loc[0].formattedAddress,
    street: loc[0].streetName,
    city: loc[0].city,
    state: loc[0].stateCode,
    zipcode: loc[0].zipcode,
    country: loc[0].countryCode
  };

  // Do not save address in DB
  this.address = undefined;
  next();
});

// Cascade delete courses when a bootcamp is deleted
BootcampSchema.pre('remove', async function(next) {
  console.log(`Courses being removed from bootcamp ${this._id}`);
  await this.model('Course').deleteMany({ bootcamp: this._id });
  next();
});

// Reverse populate with virtuals
BootcampSchema.virtual('courses', {
  ref: 'Course',
  localField: '_id',
  foreignField: 'bootcamp',
  justOne: false
});

module.exports = mongoose.model('Bootcamp', BootcampSchema);
```

### Create the above model
```js
const Bootcamp = require('../models/Bootcamp');

// Create
const bootcamp = await Bootcamp.create(data); // return the created object

// GET
const bootcamps = await Bootcamp.find(); // find all the bootcamps

// Single
const bootcamps = await Bootcamp.findById(id); // find specific bootcamp

// find and update 
// data can be partial, means let say from above model we want to update career from ['A'] to career ['A', 'B'], then data will be:
// {careers: ['A', 'B']}
const bootupdated = await Bootcamp.findByIdAndUpdate(id, data, {
  new: true, // return the updated data
  runValidators: true
});

// delete
const bootdeleted = await Bootcamp.findByIdAndDelete(id);

```

### Mongoose Bad error, expresserrorHandler:
```js
const ErrorResponse = require('../utils/errorResponse');

const errorHandler = (err, req, res, next) => {
  let error = { ...err };

  error.message = err.message;

  // Log to console for dev
  console.log(err);

  // Mongoose bad ObjectId, when the id is not properlly formated, while findById
  if (err.name === 'CastError') {
    const message = `Resource not found`;
    error = new ErrorResponse(message, 404);
  }

  // Mongoose duplicate key
  if (err.code === 11000) {
    const message = 'Duplicate field value entered';
    error = new ErrorResponse(message, 400);
  }

  // Mongoose validation error
  if (err.name === 'ValidationError') {
    const message = Object.values(err.errors).map(val => val.message);
    error = new ErrorResponse(message, 400);
  }

  res.status(error.statusCode || 500).json({
    success: false,
    error: error.message || 'Server Error'
  });
};

module.exports = errorHandler;
```




















