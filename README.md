# TrabajoPractico

ESTUDIANTE:
SERGIO NILO URQUIDI CALDERON

DATOS QUE TIENE EL CODIGO
Autenticación con JWT
Gestión de usuarios con estados activo/inactivo
CRUD completo para tareas
Relaciones entre usuarios y tareas
Manejo de errores
Middleware de autenticación
Validaciones de datos
Migraciones de base de datos

LIBRERÍAS 
npm init -y
npm install express sequelize pg pg-hstore cors jsonwebtoken bcrypt dotenv

DATOS DE CONEXIÓN
PORT=3000
JWT_SECRET=tu_secreto_jwt_seguro
DATABASE_URL=postgres://usuario:contraseña@localhost:5432/tasks_db
NODE_ENV=development

PROYECTO RESULTO: CÓDIGO FUENTE

// package.json
{
  "name": "tasks-api",
  "version": "1.0.0",
  "description": "TRABAJO API",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  },
  "dependencies": {
    "bcrypt": "^5.1.0",
    "cors": "^2.8.5",
    "dotenv": "^16.0.3",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0",
    "pg": "^8.10.0",
    "pg-hstore": "^2.3.4",
    "sequelize": "^6.31.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}

// .env
PORT=3000
JWT_SECRET=your_jwt_secret_key
DATABASE_URL=postgres://user:password@localhost:5432/tasks_db

// src/server.js
require('dotenv').config();
const app = require('./app');
const { sequelize } = require('./models');

const PORT = process.env.PORT || 3000;

async function startServer() {
  try {
    await sequelize.authenticate();
    console.log('Database connection has been established successfully.');
    
    app.listen(PORT, () => {
      console.log(`Server is running on port ${PORT}`);
    });
  } catch (error) {
    console.error('Unable to connect to the database:', error);
  }
}

startServer();

// src/config/config.js
require('dotenv').config();

module.exports = {
  development: {
    username: "postgres",
    password: "your_password",
    database: "tasks_db",
    host: "127.0.0.1",
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

// src/models/index.js
const fs = require('fs');
const path = require('path');
const Sequelize = require('sequelize');
const process = require('process');
const basename = path.basename(__filename);
const env = process.env.NODE_ENV || 'development';
const config = require(__dirname + '/../config/config.js')[env];
const db = {};

let sequelize;
if (config.use_env_variable) {
  sequelize = new Sequelize(process.env[config.use_env_variable], config);
} else {
  sequelize = new Sequelize(config.database, config.username, config.password, config);
}

fs
  .readdirSync(__dirname)
  .filter(file => {
    return (
      file.indexOf('.') !== 0 &&
      file !== basename &&
      file.slice(-3) === '.js' &&
      file.indexOf('.test.js') === -1
    );
  })
  .forEach(file => {
    const model = require(path.join(__dirname, file))(sequelize, Sequelize.DataTypes);
    db[model.name] = model;
  });

Object.keys(db).forEach(modelName => {
  if (db[modelName].associate) {
    db[modelName].associate(db);
  }
});

db.sequelize = sequelize;
db.Sequelize = Sequelize;

module.exports = db;

// src/models/user.js
const { Model } = require('sequelize');
const bcrypt = require('bcrypt');

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
      unique: true,
      validate: {
        notEmpty: true
      }
    },
    password: {
      type: DataTypes.STRING,
      allowNull: false,
      validate: {
        notEmpty: true
      }
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
      allowNull: false,
      validate: {
        notEmpty: true
      }
    },
    done: {
      type: DataTypes.BOOLEAN,
      defaultValue: false
    },
    userId: {
      type: DataTypes.INTEGER,
      allowNull: false,
      references: {
        model: 'Users',
        key: 'id'
      }
    }
  }, {
    sequelize,
    modelName: 'Task',
  });
  
  return Task;
};

// src/middlewares/auth.middleware.js
const jwt = require('jsonwebtoken');
const { User } = require('../models');

const authMiddleware = async (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({ message: 'Authorization token required' });
    }

    const token = authHeader.split(' ')[1];
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    const user = await User.findByPk(decoded.id);
    if (!user || user.status === 'inactive') {
      return res.status(401).json({ message: 'User not found or inactive' });
    }

    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ message: 'Invalid token' });
  }
};

module.exports = authMiddleware;

// src/middlewares/error.middleware.js
const errorHandler = (err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ message: 'Something went wrong!' });
};

module.exports = errorHandler;

// src/controllers/user.controller.js
const { User, Task } = require('../models');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');

const userController = {
  // Get all users
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

  // Get user by ID
  async getUserById(req, res) {
    try {
      const user = await User.findByPk(req.params.id, {
        attributes: ['id', 'username', 'status']
      });
      
      if (!user) {
        return res.status(404).json({ message: 'User not found' });
      }
      
      res.json(user);
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  },

  // Create user
  async createUser(req, res) {
    try {
      const { username, password } = req.body;
      
      if (!username || !password) {
        return res.status(400).json({ message: 'Username and password are required' });
      }

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
      if (error.name === 'SequelizeUniqueConstraintError') {
        return res.status(400).json({ message: 'Username already exists' });
      }
      res.status(400).json({ message: error.message });
    }
  },

  // Update user
  async updateUser(req, res) {
    try {
      const { username, password } = req.body;
      const user = await User.findByPk(req.params.id);
      
      if (!user) {
        return res.status(404).json({ message: 'User not found' });
      }

      const updates = {};
      if (username) updates.username = username;
      if (password) updates.password = await bcrypt.hash(password, 10);

      await user.update(updates);
      
      res.json({
        id: user.id,
        username: user.username,
        status: user.status
      });
    } catch (error) {
      res.status(400).json({ message: error.message });
    }
  },

  // Update user status
  async updateUserStatus(req, res) {
    try {
      const { status } = req.body;
      
      if (!['active', 'inactive'].includes(status)) {
        return res.status(400).json({ message: 'Invalid status value' });
      }

      const user = await User.findByPk(req.params.id);
      
      if (!user) {
        return res.status(404).json({ message: 'User not found' });
      }

      await user.update({ status });
      
      res.json({
        id: user.id,
        username: user.username,
        status: user.status
      });
    } catch (error) {
      res.status(400).json({ message: error.message });
    }
  },

  // Delete user
  async deleteUser(req, res) {
    try {
      const user = await User.findByPk(req.params.id);
      
      if (!user) {
        return res.status(404).json({ message: 'User not found' });
      }

      await user.destroy();
      res.status(204).send();
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  },

  // Login
  async login(req, res) {
    try {
      const { username, password } = req.body;
      
      if (!username || !password) {
        return res.status(400).json({ message: 'Username and password are required' });
      }

      const user = await User.findOne({ where: { username } });
      
      if (!user || !(await bcrypt.compare(password, user.password))) {
        return res.status(401).json({ message: 'Invalid credentials' });
      }

      if (user.status === 'inactive') {
        return res.status(401).json({ message: 'User is inactive' });
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

  // Get user tasks
  async getUserTasks(req, res) {
    try {
      const user = await User.findByPk(req.params.id, {
        include: [{
          model: Task,
          as: 'tasks'
        }]
      });
      
      if (!user) {
        return res.status(404).json({ message: 'User not found' });
      }
      
      res.json(user.tasks);
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  }
};

module.exports = userController;

// src/controllers/task.controller.js
const { Task } = require('../models');

const taskController = {
  // Get all tasks for current user
  async getAllTasks(req, res) {
    try {
      const tasks = await Task.findAll({
        where: { userId: req.user.id }
      });
      res.json(tasks);
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  },

  // Get task by ID
  async getTaskById(req, res) {
    try {
      const task = await Task.findOne({
        where: {
          id: req.params.id,
          userId: req.user.id
        }
      });
      
      if (!task) {
        return res.status(404).json({ message: 'Task not found' });
      }
      
      res.json(task);
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  },

  // Create task
  async createTask(req, res) {
    try {
      const { name } = req.body;
      
      if (!name) {
        return res.status(400).json({ message: 'Task name is required' });
      }

      const task = await Task.create({
        name,
        userId: req.user.id
      });
      
      res.status(201).json(task);
    } catch (error) {
      res.status(400).json({ message: error.message });
    }
  },

  // Update task
  async updateTask(req, res) {
    try {
      const { name } = req.body;
      const task = await Task.findOne({
        where: {
          id: req.params.id,
          userId: req.user.id
        }
      });
      
      if (!task) {
        return res.status(404).json({ message: 'Task not found' });
      }

      await task.update({ name });
      res.json(task);
    } catch (error) {
      res.status(400).json({ message: error.message });
    }
  },

  // Update task status
  async updateTaskStatus(req, res) {
    try {
      const { done } = req.body;
      
      if (typeof done !== 'boolean') {
        return res.status(400).json({ message: 'Done must be a boolean value' });
      }

      const task = await Task.findOne({
        where: {
          id: req.params.id,
          userId: req.user.id
        }
      });
      
      if (!task) {
        return res.status(404).json({ message: 'Task not found' });
      }

      await task.update({ done });
      res.json(task);
    } catch (error) {
      res.status(400).json({ message:

      res.status(400).json({ message: error.message });
    }
  },

  // Delete task
  async deleteTask(req, res) {
    try {
      const task = await Task.findOne({
        where: {
          id: req.params.id,
          userId: req.user.id
        }
      });
      
      if (!task) {
        return res.status(404).json({ message: 'Task not found' });
      }

      await task.destroy();
      res.status(204).send();
    } catch (error) {
      res.status(500).json({ message: error.message });
    }
  }
};

module.exports = taskController;

// src/routes/index.js

const express = require('express');
const router = express.Router();
const userRoutes = require('./user.routes');
const taskRoutes = require('./task.routes');

router.use('/users', userRoutes);
router.use('/tasks', taskRoutes);

module.exports = router;

// src/routes/user.routes.js

const express = require('express');
const router = express.Router();
const userController = require('../controllers/user.controller');
const authMiddleware = require('../middlewares/auth.middleware');

// Rutas públicas

router.post('/', userController.createUser);
router.get('/', userController.getAllUsers);
router.post('/login', userController.login);

// Rutas protegidas
router.use(authMiddleware);
router.get('/:id', userController.getUserById);
router.put('/:id', userController.updateUser);
router.patch('/:id', userController.updateUserStatus);
router.delete('/:id', userController.deleteUser);
router.get('/:id/tasks', userController.getUserTasks);

module.exports = router;

// src/routes/task.routes.js

const express = require('express');
const router = express.Router();
const taskController = require('../controllers/task.controller');
const authMiddleware = require('../middlewares/auth.middleware');

router.use(authMiddleware);

router.get('/', taskController.getAllTasks);
router.post('/', taskController.createTask);
router.get('/:id', taskController.getTaskById);
router.put('/:id', taskController.updateTask);
router.patch('/:id', taskController.updateTaskStatus);
router.delete('/:id', taskController.deleteTask);

module.exports = router;

// src/app.js

const express = require('express');
const cors = require('cors');
const routes = require('./routes');
const errorHandler = require('./middlewares/error.middleware');

const app = express();

// Middlewares
app.use(cors());
app.use(express.json());

// Routes
app.use('/api', routes);

// Error handling
app.use(errorHandler);

// Not found handler
app.use((req, res) => {
  res.status(404).json({ message: 'Route not found' });
});

module.exports = app;

// migrations/YYYYMMDDHHMMSS-create-users.js

'use strict';

module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('Users', {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.INTEGER
      },
      username: {
        type: Sequelize.STRING,
        allowNull: false,
        unique: true
      },
      password: {
        type: Sequelize.STRING,
        allowNull: false
      },
      status: {
        type: Sequelize.ENUM('active', 'inactive'),
        defaultValue: 'active'
      },
      createdAt: {
        allowNull: false,
        type: Sequelize.DATE
      },
      updatedAt: {
        allowNull: false,
        type: Sequelize.DATE
      }
    });
  },
  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('Users');
  }
};

// migrations/YYYYMMDDHHMMSS-create-tasks.js

'use strict';

module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('Tasks', {
      id: {
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
        type: Sequelize.INTEGER
      },
      name: {
        type: Sequelize.STRING,
        allowNull: false
      },
      done: {
        type: Sequelize.BOOLEAN,
        defaultValue: false
      },
      userId: {
        type: Sequelize.INTEGER,
        allowNull: false,
        references: {
          model: 'Users',
          key: 'id'
        },
        onUpdate: 'CASCADE',
        onDelete: 'CASCADE'
      },
      createdAt: {
        allowNull: false,
        type: Sequelize.DATE
      },
      updatedAt: {
        allowNull: false,
        type: Sequelize.DATE
      }
    });
  },
  down: async (queryInterface, Sequelize) => {
    await queryInterface.dropTable('Tasks');
  }
};

