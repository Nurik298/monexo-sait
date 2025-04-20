// Backend логикасы (Node.js + Express + MongoDB) + Admin Panel

const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const app = express();

app.use(cors());
app.use(express.json());

mongoose.connect('mongodb://localhost:27017/monexo', {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

// --- Models ---
const UserSchema = new mongoose.Schema({
  email: String,
  name: String,
  profilePic: String,
  userId: String,
  invitedBy: String,
  balance: { type: Number, default: 0 },
  watchedVideos: [String],
  quizAnswers: Object,
  joinedLotteries: [String],
  withdrawalRequests: [
    {
      bankName: String,
      account: String,
      status: { type: String, default: 'pending' },
    },
  ],
});

const User = mongoose.model('User', UserSchema);

const QuizSchema = new mongoose.Schema({
  title: String,
  questions: [
    {
      question: String,
      options: [String],
      correct: String,
      timeLimit: Number,
    },
  ],
});

const Quiz = mongoose.model('Quiz', QuizSchema);

const AdminSchema = new mongoose.Schema({
  username: String,
  password: String, // (hash қолдану керек нақтыда)
});

const Admin = mongoose.model('Admin', AdminSchema);

// --- Routes ---

app.post('/api/user', async (req, res) => {
  const { email, name, profilePic, invitedBy } = req.body;
  let user = await User.findOne({ email });
  if (!user) {
    const userId = Math.random().toString(36).substr(2, 10);
    user = new User({ email, name, profilePic, invitedBy, userId });
    await user.save();
  }
  res.json(user);
});

app.post('/api/watch', async (req, res) => {
  const { userId, videoId } = req.body;
  const user = await User.findOne({ userId });
  if (!user.watchedVideos.includes(videoId)) {
    user.watchedVideos.push(videoId);
    user.balance += 300;
    await user.save();
  }
  res.json(user);
});

app.post('/api/quiz/submit', async (req, res) => {
  const { userId, answers } = req.body;
  const user = await User.findOne({ userId });
  user.quizAnswers = answers;
  user.balance += 1000;
  await user.save();
  res.json(user);
});

app.post('/api/lottery/join', async (req, res) => {
  const { userId, lotteryId } = req.body;
  const user = await User.findOne({ userId });
  if (!user.joinedLotteries.includes(lotteryId)) {
    user.joinedLotteries.push(lotteryId);
    await user.save();
  }
  res.json(user);
});

app.post('/api/withdraw', async (req, res) => {
  const { userId, bankName, account } = req.body;
  const user = await User.findOne({ userId });
  user.withdrawalRequests.push({ bankName, account });
  await user.save();
  res.json({ message: 'Сұраныс қабылданды' });
});

// --- Admin Panel Routes ---

// Барлық қолданушыларды көру
app.get('/api/admin/users', async (req, res) => {
  const users = await User.find();
  res.json(users);
});

// Төлем сұраныстарын көру
app.get('/api/admin/withdrawals', async (req, res) => {
  const users = await User.find({ 'withdrawalRequests.0': { $exists: true } });
  const allRequests = users.flatMap(user => user.withdrawalRequests.map(r => ({
    userId: user.userId,
    email: user.email,
    bankName: r.bankName,
    account: r.account,
    status: r.status,
  })));
  res.json(allRequests);
});

// Төлем сұранысын бекіту/бас тарту
app.post('/api/admin/withdraw/update', async (req, res) => {
  const { userId, account, status } = req.body;
  const user = await User.findOne({ userId });
  const request = user.withdrawalRequests.find(r => r.account === account);
  if (request) request.status = status;
  await user.save();
  res.json({ message: 'Жаңартылды' });
});

app.listen(5000, () => {
  console.log('Backend running on http://localhost:5000');
});

