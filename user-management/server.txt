const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");

const app = express();

app.use(cors());
app.use(express.json());

mongoose
.connect("mongodb+srv://20239673:20239673@cluster0.jrmoady.mongodb.net/it4409?appName=Cluster0")
.then(() => console.log("Connected to MongoDB"))
.catch((err) => console.error("MongoDB Error:", err));

const UserSchema = new mongoose.Schema({
name: {
    type: String,
    required: [true, "Tên không được để trống"],
    minlength: [2, "Tên phải có ít nhất 2 ký tự"],
},
age: {
    type: Number,
    required: [true, "Tuổi không được để trống"],
    min: [0, "Tuổi phải >= 0"],
},
email: {
    type: String,
    required: [true, "Email không được để trống"],
    match: [/^\S+@\S+\.\S+$/, "Email không hợp lệ"],
},
address: {
    type: String,
},
});

const User = mongoose.model("User", UserSchema);

// GET: Lấy danh sách user có phân trang + tìm kiếm
app.get("/api/users", async (req, res) => {
try {
    // Giới hạn page >= 1, limit trong khoảng [1, 50]
    const page = Math.max(1, parseInt(req.query.page) || 1);
    const limit = Math.min(50, Math.max(1, parseInt(req.query.limit) || 5));
    const search = req.query.search || "";

    const filter = search
    ? {
        $or: [
            { name: { $regex: search, $options: "i" } },
            { email: { $regex: search, $options: "i" } },
            { address: { $regex: search, $options: "i" } },
        ],
        }
    : {};

    const skip = (page - 1) * limit;

    // Promise.all để chạy song song 2 truy vấn
    const [users, total] = await Promise.all([
    User.find(filter).skip(skip).limit(limit),
    User.countDocuments(filter),
    ]);

    res.json({
    page,
    limit,
    total,
    totalPages: Math.ceil(total / limit),
    data: users,
    });
} catch (err) {
    res.status(500).json({ error: err.message });
}
});

// POST: Tạo user mới
app.post("/api/users", async (req, res) => {
try {
    // Chuẩn hóa đầu vào
    const name = req.body.name?.trim();
    const age = req.body.age !== undefined ? Math.floor(Number(req.body.age)) : undefined;
    const email = req.body.email?.trim().toLowerCase();
    const address = req.body.address?.trim();

    // Kiểm tra email đã tồn tại chưa
    const existing = await User.findOne({ email });
    if (existing) return res.status(400).json({ error: "Email đã tồn tại" });

    const newUser = await User.create({ name, age, email, address });
    res.status(201).json({ message: "Tạo người dùng thành công", data: newUser });
} catch (err) {
    res.status(400).json({ error: err.message });
}
});

// PUT: Cập nhật user - chỉ cập nhật trường được truyền vào
app.put("/api/users/:id", async (req, res) => {
try {
    // Kiểm tra ID hợp lệ
    if (!mongoose.Types.ObjectId.isValid(req.params.id))
    return res.status(400).json({ error: "ID không hợp lệ" });

    // Chỉ đưa vào updateData những trường thực sự được gửi lên
    const updateData = {};
    if (req.body.name !== undefined) updateData.name = req.body.name.trim();
    if (req.body.age !== undefined) updateData.age = Math.floor(Number(req.body.age));
    if (req.body.email !== undefined) updateData.email = req.body.email.trim().toLowerCase();
    if (req.body.address !== undefined) updateData.address = req.body.address.trim();

    const updatedUser = await User.findByIdAndUpdate(
    req.params.id,
    updateData,
    { new: true, runValidators: true }
    );

    if (!updatedUser) return res.status(404).json({ error: "Không tìm thấy người dùng" });
    res.json({ message: "Cập nhật người dùng thành công", data: updatedUser });
} catch (err) {
    res.status(400).json({ error: err.message });
}
});

// DELETE: Xóa user
app.delete("/api/users/:id", async (req, res) => {
try {
    // Kiểm tra ID hợp lệ trước khi xóa
    if (!mongoose.Types.ObjectId.isValid(req.params.id))
    return res.status(400).json({ error: "ID không hợp lệ" });

    const deletedUser = await User.findByIdAndDelete(req.params.id);
    if (!deletedUser) return res.status(404).json({ error: "Không tìm thấy người dùng" });
    res.json({ message: "Xóa người dùng thành công" });
} catch (err) {
    res.status(400).json({ error: err.message });
}
});

app.listen(3001, () => {
console.log("Server running on http://localhost:3001");
});