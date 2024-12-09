# TrabajoPractico

ESTUDIANTE:
SERGIO NILO URQUIDI CALDERON

LIBRERÍAS 
npm init -y
npm install express sequelize pg pg-hstore cors jsonwebtoken bcrypt dotenv

DATOS DE CONEXIÓN
DATABASE_URL=tu_url_de_postgres
JWT_SECRET=tu_secreto_jwt
PORT=3000

PROYECTO RESULTO: CÓDIGO FUENTE

// src/config/config.js
module.exports = {
  development: {
    username: "postgres",
    password: "your_password",
    database: "tasks_db",
    host: "localhost",
    dialect: "postgres"
  },
  production: {
    use_env_variable: "DATABASE_URL",
    dialect: "postgres",
    dialectOptions: {
      ssl: {
        require: true,
        rejectUnauthorized: false
      }
    }
  }
};

// src/models/user.js
const { Model } = require('sequelize');

module.exports = (sequelize, DataTypes) => {
  class User extends Model {
    static associate(models) {
      User.hasMany(models.Task, {
        foreignKey: 'userId',
        as: 'tasks'
      });
    }
  }
  
  User.init({
    username: {
      type: DataTypes.STRING,
      allowNull: false,
      unique: true
    },
    password: {
      type: DataTypes.STRING,
      allowNull: false
    },
    status: {
      type: DataTypes.ENUM('active', 'inactive'),
      defaultValue: 'active'
    }
  }, {
    sequelize,
    modelName: 'User',
  });
  
  return User;
};

// src/models/task.js
const { Model } = require('sequelize');

module.exports = (sequelize, DataTypes) => {
  class Task extends Model {
    static associate(models) {
      Task.belongsTo(models.User, {
        foreignKey: 'userId',
        as: 'user'
      });
    }
  }
  
  Task.init({
    name: {
      type: DataTypes.STRING,
      allowNull: false
    },
    done: {
      type: DataTypes.BOOLEAN,
      defaultValue: false
    },
    userId: {
      type: DataTypes.INTEGER,
      allowNull: false
    }
  }, {
    sequelize,
    modelName: 'Task',
  });
  
  return Task;
};


// src/app.js
const express = require('express');
const cors = require('cors');
const routes = require('./routes');

const app = express();

app.use(cors());
app.use(express.json());
app.use('/api', routes);

module.exports = app;

// src/middleware/auth.js
const jwt = require('jsonwebtoken');

const authMiddleware = (req, res, next) => {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) {
      return res.status(401).json({ message: 'No token provided' });
    }
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid token' });
  }
};

module.exports = authMiddleware;

// src/routes/index.js
const express = require('express');
const router = express.Router();
const userRoutes = require('./user.routes');
const taskRoutes = require('./task.routes');
const authMiddleware = require('../middleware/auth');

// Rutas públicas
router.use('/users', userRoutes.publicRoutes);
router.use('/login', userRoutes.loginRoute);

// Rutas protegidas
router.use('/users', authMiddleware, userRoutes.protectedRoutes);
router.use('/tasks', authMiddleware, taskRoutes);

module.exports = router;

// src/controllers/user.controller.js
const { User } = require('../models');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');

const userController = {
  async getAllUsers(req, res) {
    try {
      const users = await User.findAll({
        attributes: ['id', 'username', 'status']
      });
      res.json(users);
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  },

  async createUser(req, res) {
    try {
      const { username, password } = req.body;
      const hashedPassword = await bcrypt.hash(password, 10);
      const user = await User.create({
        username,
        password: hashedPassword
      });
      res.status(201).json({
        id: user.id,
        username: user.username,
        status: user.status
      });
    } catch (error) {
      res.status(400).json({ message: error.message });
    }
  },

  async login(req, res) {
    try {
      const { username, password } = req.body;
      const user = await User.findOne({ where: { username } });
      
      if (!user || !(await bcrypt.compare(password, user.password))) {
        return res.status(401).json({ message: 'Invalid credentials' });
      }
      
      const token = jwt.sign(
        { id: user.id, username: user.username },
        process.env.JWT_SECRET,
        { expiresIn: '24h' }
      );
      
      res.json({ token });
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  },

  // ... otros métodos del controlador
};

module.exports = userController;

// src/controllers/task.controller.js
const { Task } = require('../models');

const taskController = {
  async getUserTasks(req, res) {
    try {
      const tasks = await Task.findAll({
        where: { userId: req.user.id }
      });
      res.json(tasks);
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  },

  async createTask(req, res) {
    try {
      const { name } = req.body;
      const task = await Task.create({
        name,
        userId: req.user.id
      });
      res.status(201).json(task);
    } catch (error) {
      res.status(400).json({ message: error.message });
    }
  },

  // ... otros métodos del controlador
};

module.exports = taskController;
