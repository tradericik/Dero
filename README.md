
chmod +x create_ai_sales_project.sh
./create_ai_sales_project.sh
#!/usr/bin/env bash
set -e

# create_ai_sales_project.sh
# Creates an ai-sales project (backend + frontend + CI workflow files)
# Run this script in an empty directory. It will create ./ai-sales

ROOT="ai-sales"
if [ -d "$ROOT" ]; then
  echo "Folder $ROOT already exists. Delete or move it and re-run."
  exit 1
fi

mkdir -p "$ROOT"
cd "$ROOT"

####################
# BACKEND
####################
mkdir -p backend/src/{models,routes,services,jobs}
cat > backend/package.json <<'JSON'
{
  "name": "ai-sales-backend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^7.0.4",
    "dotenv": "^16.0.3",
    "bullmq": "^1.97.0",
    "ioredis": "^5.3.2",
    "openai": "^4.10.0",
    "@sendgrid/mail": "^7.8.0",
    "jsonwebtoken": "^9.0.0",
    "bcrypt": "^5.1.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
JSON

cat > backend/.env.example <<'ENV'
PORT=4000
MONGO_URI=mongodb://localhost:27017/ai-sales
REDIS_URL=redis://127.0.0.1:6379
JWT_SECRET=change_me
OPENAI_API_KEY=sk-...
SENDGRID_API_KEY=SG...
SENDER_EMAIL=no-reply@yourdomain.com
ENV

cat > backend/src/server.js <<'JS'
const express = require('express');
const mongoose = require('mongoose');
const dotenv = require('dotenv');
dotenv.config();
const app = express();
app.use(express.json());

// simple health
app.get('/api/health', (req, res) => res.json({ ok: true }));

// mount routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/leads', require('./routes/leads'));
app.use('/api/campaigns', require('./routes/campaigns'));
app.use('/api/webhooks', require('./routes/webhooks'));

// init reply worker (background)
try {
  require('./jobs/processReplyJob').initReplyWorker && require('./jobs/processReplyJob').initReplyWorker();
} catch(e) {
  console.log('Reply worker not initialized (ok for dev).', e.message || e);
}

const PORT = process.env.PORT || 4000;
mongoose.connect(process.env.MONGO_URI || 'mongodb://localhost:27017/ai-sales')
  .then(()=> {
    console.log('Mongo connected');
    app.listen(PORT, ()=> console.log('Server listening on', PORT));
  }).catch(err => console.error(err));
JS

# models
cat > backend/src/models/User.js <<'JS'
const mongoose = require('mongoose');
const UserSchema = new mongoose.Schema({
  name: String,
  email: { type: String, index: true, unique: true },
  passwordHash: String
}, { timestamps: true });
module.exports = mongoose.model('User', UserSchema);
JS

cat > backend/src/models/Lead.js <<'JS'
const mongoose = require('mongoose');
const LeadSchema = new mongoose.Schema({
  ownerId: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  firstName: String,
  lastName: String,
  email: { type: String, index: true },
  company: String,
  tags: [String],
  status: { type: String, default: 'new' }
}, { timestamps: true });
module.exports = mongoose.model('Lead', LeadSchema);
JS

cat > backend/src/models/Campaign.js <<'JS'
const mongoose = require('mongoose');
const CampaignSchema = new mongoose.Schema({
  ownerId: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  name: String,
  steps: [mongoose.Schema.Types.Mixed],
  isActive: { type: Boolean, default: false }
}, { timestamps: true });
module.exports = mongoose.model('Campaign
