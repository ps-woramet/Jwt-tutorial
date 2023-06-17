0. .gitignore
    node_modules

1. ติดตั้ง package.json
    npm init -y

2. สร้างไฟล์
    config -> database.js
    middleware -> auth.js
    model -> user.js
    app.js
    index.js

3. ติดตั้ง dependencies

    npm i mongoose express jsonwebtoken dotenv bcryptjs

4. ติดตั้ง dependencies แบบ dev

    npm i nodemon -D

5. Create nodejs server and connect to database

    5.1 database.js ทำการเชื่อมต่อ database

        const mongoose = require('mongoose');

        const {MONGO_URI} = process.env;

        exports.connect = () => {

            // Connecting to database

            mongoose.connect(MONGO_URI,{
                useNewUrlParser: true,
                // userUnifiedTopology: true,
                // useCreateIndex: true,
                // useGindAndModify: false
            })
            .then(() => {
                console.log("Successfully connected to database");
            })
            .catch((error) => {
                console.log("Error connecting to database");
                console.error(error);
                process.exit(1);
            })
        }

    5.2 -app.js ทำการสร้างหน้าสำหรับ login

        require('dotenv').config();
        require('./config/database').connect();

        const express = require('express');

        const app = express();

        app.use(express.json());

        // Login here

        module.exports = app;

    5.3 index.js ทำการสร้าง server

        const http = require('http');
        const app = require('./app');
        const server = http.createServer(app);

        const {API_PORT} = process.env;
        const port = process.env.PORT || API_PORT;

        // Server listening
        server.listen(port, () => {
            console.log(`Server running on port ${port}`);
        })

    5.4 ทำการสร้างฐานข้อมูลที่ https://cloud.mongodb.com

        ทำการ edit password (ไปที่ database access -> edit)และ นำ url สำหรับใช้เชื่อมต่อมา

        mongodb+srv://admin:<password>@dipeshcluster.xzprbdf.mongodb.net/

    5.5 สร้างไฟล์ .env

        API_PORT = 4001
        MONGO_URI = mongodb+srv://admin:a123456@dipeshcluster.xzprbdf.mongodb.net/

    5.6 package.json ทำการแก้ script

        "scripts": {
            "start": "node index.js",
            "dev": "nodemon index.js"
        }

    5.7 terminal -> npm run dev

6. Create user model

    6.1 ทำการสร้าง userSchema เมื่อมี user ใหม่

        const mongoose = require('mongoose');

        const userSchema = new mongoose.Schema({
            first_name: {type: String, default: null},
            last_name: {type: String, default: null},
            email: {type: String, unique: true},
            password: {type: String},
            token: {type: String}
        })

        module.exports = mongoose.model('user', userSchema)

7. create the routes for register and login
    
    7.1 .env เพิ่ม TOKEN_KEY

        API_PORT = 4001
        MONGO_URI = mongodb+srv://admin:a123456@dipeshcluster.xzprbdf.mongodb.net/
        TOKEN_KEY = qwertyui

    7.2 app.js ทำการเพิ่ม route (register) และ require bcryptjs, jsonwebtoken, user(model -> user.js)

        const User = require('./model/user');
        const bcrypt = require('bcryptjs');
        const jwt = require('jsonwebtoken');

        // Register

        app.post("/register", async (req, res) => {
            // our register logic gose here

            try{
                // Get user input
                const {first_name, last_name, email, password} = req.body;
                
                // Valdate user input
                if (!(email && password && first_name && last_name)){
                    res.status(400).send("All input is requried");
                }

                // Check if user already exist
                //Validate if user exist in our database
                const oldUser = await User.findOne({email});

                if (oldUser){
                    return res.status(400).send("User already exist. Please login")
                }

                // Encrypt user password
                encryptedPassword = await bcrypt.hash(password, 10);

                // Create user in our database
                const user = await User.create({
                    first_name,
                    last_name,
                    email: email.toLowerCase(),
                    password: encryptedPassword
                })

                // Create token
                const token = jwt.sign(
                    {user_id: user._id, email},
                    process.env.TOKEN_KEY,{
                        expiresIn: "2h"
                    }
                )

                // Save user token
                user.token = token;

                // return new user
                res.status(201).json(user);
            
            }catch (err){
                console.log(err);
            }
        })
        
    7.3 postman

        post http://localhost:4001/register

        {
            "first_name": "woramet",
            "last_name": "tompudsa",
            "email": "ps.woramet@gmail.com",
            "password": "123456"
        }

    7.4 app.js ทำการเพิ่ม route (login)

        // Login
        app.post("/login", async (req, res) => {
            // our login logic gose here

            try{
                // Get user input
                const {email, password} =req.body;

                // Validate user input
                if(!(email && password)){
                    res.status(400).send("All input is required")
                }

                // Validate if user exist in our database
                const user = await User.findOne({email});

                if(user && (await bcrypt.compare(password, user.password))){
                
                    // Create token
                    const token = jwt.sign(
                        {user_id: user._id, email},
                        process.env.TOKEN_KEY,
                        {
                            expiresIn: "2h"
                        }
                    )

                    // Save user token
                    user.token = token;

                    res.status(200).json(user);
                }

                res.status(400).send("Invalid Credentials")
            }
            catch(err){
                console.log(err);
            }
        })
    
    7.5 postman
        
        post http://localhost:4001/login

        {
            "email": "ps.woramet@gmail.com",
            "password": "123456"
        }

8. create middleware for authentication

    8.1 auth.js ทำการสร้าง middleware สำหรับ authentication

        const jwt = require('jsonwebtoken');

        const config = process.env;

        const verifyToken = (req, res, next) => {
            const token = req.body.token || req.query.token || req.headers['x-access-token'];

            if (!token){
                return res.status(403).send("A token is required for authentication")
            }

            try {
                const decoded = jwt.verify(token, config.TOKEN_KEY);
                req.user = decoded;
            } catch(err){
                return res.status(401).send("Invalid Token")
            }

            return next();
        }

        module.exports = verifyToken;

    8.2 app.js ทำการเพิ่ม route (welcome) และ require auth(middleware -> auth.js)


        const auth = require('./middleware/auth');

        app.post('/welcome', auth, (req, res) => {
        res.status(200).send('Welcome');
        })

    8.3 postman

        post http://localhost:4001/welcome

        Headers

        x-access-token : "ใส่ค่า token จากการ login"