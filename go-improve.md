## 不应该抛给上层
## 原因：

对于上层来说，并不需要知道ErrNoRows，上层只关心查询过程中是否产生错误，如果是空的话，应该返回空slice，空map或nil。