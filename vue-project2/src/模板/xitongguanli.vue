<template>
  <div class="profile-center">
    <div class="header-section card">
      <div class="avatar-section">
        <img src="https://via.placeholder.com/100x100" alt="头像" class="avatar">
        <div class="user-info">
          <h2 class="username">用户名</h2>
          <p class="user-role">角色：管理员</p>
        </div>
      </div>
    </div>

    <div class="settings-grid">
      <div class="setting-category card">
        <div class="category-header">
          <h3>个人信息</h3>
          <p>查看和编辑个人资料</p>
        </div>
        <div class="category-actions">
          <button class="btn">编辑资料</button>
        </div>
      </div>

      <div class="setting-category card">
        <div class="category-header">
          <h3>账户安全</h3>
          <p>修改密码和切换账户</p>
        </div>
        <div class="category-actions">
          <button class="btn" @click="showChangePasswordModal">修改密码</button>
          <button class="btn btn-secondary" @click="showSwitchAccountModal">切换账户</button>
        </div>
      </div>

      <div class="setting-category card">
        <div class="category-header">
          <h3>个人设置</h3>
          <p>个性化和偏好设置</p>
        </div>
        <div class="category-actions">
          <button class="btn">外观设置</button>
          <button class="btn btn-secondary">语言设置</button>
        </div>
      </div>

      <div class="setting-category card">
        <div class="category-header">
          <h3>我的收藏</h3>
          <p>查看和管理收藏内容</p>
        </div>
        <div class="category-actions">
          <button class="btn">查看收藏</button>
          <button class="btn btn-secondary">管理收藏</button>
        </div>
      </div>

      <div class="setting-category card">
        <div class="category-header">
          <h3>消息通知</h3>
          <p>设置通知偏好和接收方式</p>
        </div>
        <div class="category-actions">
          <button class="btn">通知设置</button>
          <button class="btn btn-secondary">查看历史</button>
        </div>
      </div>

      <div class="setting-category card">
        <div class="category-header">
          <h3>活动记录</h3>
          <p>查看最近的活动和操作记录</p>
        </div>
        <div class="category-actions">
          <button class="btn">查看记录</button>
          <button class="btn btn-secondary">导出记录</button>
        </div>
      </div>
    </div>

    <div class="system-actions card">
      <h3>账户操作</h3>
      <div class="actions-grid">
        <button class="action-btn danger" @click="logout">
          <i class="icon">🚪</i>
          退出登录
        </button>
        <button class="action-btn warning" @click="showAccountDeletionConfirm">
          <i class="icon">🔄</i>
          注销账户
        </button>
        <button class="action-btn primary">
          <i class="icon">📋</i>
          下载数据
        </button>
        <button class="action-btn secondary">
          <i class="icon">❓</i>
          帮助中心
        </button>
      </div>
    </div>
    
    <!-- 注销账户确认弹窗 -->
    <div v-if="showDeletionModal" class="modal-overlay" @click="closeDeletionModal">
      <div class="modal-content" @click.stop>
        <h3>确认注销账户</h3>
        <p>此操作将永久删除您的账户和所有相关数据。</p>
        <p><strong>警告：此操作无法撤销！</strong></p>
        
        <form @submit.prevent="confirmAccountDeletion">
          <div class="input-group">
            <label for="deletion-account">账号</label>
            <input 
              type="text" 
              id="deletion-account" 
              v-model="deletionAccount" 
              placeholder="请输入您的账号"
              required
            >
          </div>
          
          <div class="input-group">
            <label for="deletion-password">密码</label>
            <input 
              type="password" 
              id="deletion-password" 
              v-model="deletionPassword" 
              placeholder="请输入您的密码"
              required
            >
          </div>
          
          <div v-if="deletionError" class="error-message">
            {{ deletionError }}
          </div>
          
          <div class="modal-actions">
            <button type="button" class="btn btn-secondary" @click="closeDeletionModal">取消</button>
            <button type="submit" class="btn btn-danger">确认注销</button>
          </div>
        </form>
      </div>
    </div>
    
    <!-- 修改密码弹窗 -->
    <div v-if="showPasswordModal" class="modal-overlay" @click="closePasswordModal">
      <div class="modal-content" @click.stop>
        <h3>修改密码</h3>
        <form @submit.prevent="changePassword">
          <div class="input-group">
            <label for="current-password">当前密码</label>
            <input type="password" id="current-password" v-model="currentPassword" required>
          </div>
          
          <div class="input-group">
            <label for="new-password">新密码</label>
            <input type="password" id="new-password" v-model="newPassword" required minlength="6">
          </div>
          
          <div class="input-group">
            <label for="confirm-new-password">确认新密码</label>
            <input type="password" id="confirm-new-password" v-model="confirmNewPassword" required minlength="6">
          </div>
          
          <div class="modal-actions">
            <button type="button" class="btn btn-secondary" @click="closePasswordModal">取消</button>
            <button type="submit" class="btn btn-primary">确认修改</button>
          </div>
        </form>
      </div>
    </div>
    
    <!-- 切换账户弹窗 -->
    <div v-if="showSwitchModal" class="modal-overlay" @click="closeSwitchModal">
      <div class="modal-content" @click.stop style="width: 90%; max-width: 600px;">
        <h3>切换账户</h3>
        <p>请选择要切换到的账户:</p>
        
        <div class="account-list">
          <div 
            v-for="account in recentAccounts" 
            :key="account.id" 
            class="account-item"
            @click="switchToAccount(account)"
          >
            <div class="account-avatar">
              <img :src="account.avatar || 'https://via.placeholder.com/50x50'" :alt="account.name">
            </div>
            <div class="account-info">
              <h4>{{ account.name }}</h4>
              <p>{{ account.email }}</p>
            </div>
          </div>
        </div>
        
        <div class="divider"></div>
        
        <div class="add-account-option">
          <button class="btn btn-secondary" @click="goToAddAccount">添加新账户</button>
        </div>
        
        <div class="modal-actions">
          <button class="btn btn-secondary" @click="closeSwitchModal">取消</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import { useRouter } from 'vue-router';
import axios from 'axios';

const router = useRouter();
const showDeletionModal = ref(false);
const showPasswordModal = ref(false);
const showSwitchModal = ref(false);
const deletionError = ref('');
const deletionAccount = ref('');
const deletionPassword = ref('');

// 密码修改相关变量
const currentPassword = ref('');
const newPassword = ref('');
const confirmNewPassword = ref('');

// 最近账户列表
const recentAccounts = ref([]);

const logout = async () => {
  try {
    const token = localStorage.getItem('token');
    
    // 如果有 token，调用后端 API 清理登录状态
    if (token) {
      await axios.post('/api/auth/logout', {}, {
        headers: {
          'Authorization': `Bearer ${token}`
        }
      });
    }
    
    // 清除本地存储的认证信息
    localStorage.removeItem('token');
    
    // 跳转到登录页面
    router.push('/login');
  } catch (error) {
    console.error('退出登录失败:', error);
    // 即使后端调用失败，也要清除本地状态
    localStorage.removeItem('token');
    router.push('/login');
  }
};

const showAccountDeletionConfirm = () => {// 显示注销账户确认弹窗
  deletionError.value = '';
  deletionAccount.value = '';
  deletionPassword.value = '';
  showDeletionModal.value = true;
};

const closeDeletionModal = () => {// 关闭注销账户确认弹窗
  deletionError.value = '';
  deletionAccount.value = '';
  deletionPassword.value = '';
  showDeletionModal.value = false;
};

const confirmAccountDeletion = async () => {
  deletionError.value = '';
  
  // 验证账号和密码是否为空
  if (!deletionAccount.value || !deletionPassword.value) {
    deletionError.value = '请输入账号和密码';
    return;
  }
  
  try {
    const token = localStorage.getItem('token');
    
    // 调试输出 - 显示请求信息
    console.log('===== 注销账户请求 =====');
    console.log('账号:', deletionAccount.value);
    console.log('密码:', deletionPassword.value.substring(0, 3) + '***'); // 只显示前 3 位
    console.log('Token:', token ? token.substring(0, 20) + '...' : '不存在');
    console.log('请求 URL: /api/auth/delete-account');
    console.log('========================');
    
    // 构建请求配置
    const config = {
      headers: {}
    };
    
    // 如果 token 存在，添加认证头
    if (token) {
      config.headers['Authorization'] = `Bearer ${token}`;
    }
    
    const response = await axios.post('/api/auth/delete-account', {
      account: deletionAccount.value,
      password: deletionPassword.value
    }, config);
    
    // 清除本地存储
    localStorage.clear();
    
    // 跳转到登录页面
    router.push('/login');
  } catch (error) {
    console.error('注销账户失败:', error);
    console.error('错误状态码:', error.response?.status);
    console.error('错误数据:', error.response?.data);
    console.error('完整响应:', error.response);
    
    // 根据状态码显示具体错误信息
    if (error.response) {
      switch (error.response.status) {
        case 401:
          deletionError.value = '认证失败，请检查登录状态是否已过期';
          break;
        case 403:
          deletionError.value = '无权执行此操作';
          break;
        case 404:
          deletionError.value = '接口不存在';
          break;
        case 400:
          if (error.response.data?.message) {
            deletionError.value = error.response.data.message;
          } else if (error.response.data?.details) {
            deletionError.value = error.response.data.details.join(', ');
          } else {
            deletionError.value = '账号或密码错误';
          }
          break;
        default:
          deletionError.value = error.response.data?.message || '注销账户失败，请稍后重试';
      }
    } else if (error.request) {
      deletionError.value = '无法连接到服务器，请检查网络连接';
    } else {
      deletionError.value = error.message || '注销账户失败，请稍后重试';
    }
  } finally {
    showDeletionModal.value = false;
  }
};

// 显示修改密码弹窗
const showChangePasswordModal = () => {
  // 重置表单
  currentPassword.value = '';
  newPassword.value = '';
  confirmNewPassword.value = '';
  showPasswordModal.value = true;
};

// 关闭修改密码弹窗
const closePasswordModal = () => {
  showPasswordModal.value = false;
};

// 修改密码
const changePassword = async () => {
  // 验证新密码和确认密码是否一致
  if (newPassword.value !== confirmNewPassword.value) {
    // 不再使用 alert，可以在模板中添加错误提示
    console.error('新密码和确认密码不一致');
    return;
  }

  try {
    // TODO: 调用后端 API 修改密码
    // await api.post('/auth/change-password', {
    //   currentPassword: currentPassword.value,
    //   newPassword: newPassword.value
    // });
    
    // 关闭弹窗
    showPasswordModal.value = false;
    
    // 可以添加成功通知
    console.log('密码修改成功');
  } catch (error) {
    console.error('修改密码失败:', error);
    // 可以在 UI 中显示错误信息
    throw error;
  }
};

// 显示切换账户弹窗
const showSwitchAccountModal = async () => {
  try {
    // TODO: 调用后端 API 获取最近使用的账户列表
    // const response = await api.get('/account/recent');
    // recentAccounts.value = response.data;
    
    showSwitchModal.value = true;
  } catch (error) {
    console.error('获取账户列表失败:', error);
    // 即使获取账户列表失败，仍然显示弹窗
    showSwitchModal.value = true;
  }
};

// 关闭切换账户弹窗
const closeSwitchModal = () => {
  showSwitchModal.value = false;
};

// 切换到指定账户
const switchToAccount = async (account) => {
  try {
    // TODO: 调用后端 API 切换账户
    // await api.post('/account/switch', { accountId: account.id });
    
    // 关闭弹窗
    showSwitchModal.value = false;
    
    // 刷新页面或更新应用状态
    window.location.reload();
  } catch (error) {
    console.error('切换账户失败:', error);
    throw error;
  }
};

// 添加新账户
const goToAddAccount = () => {
  // 关闭当前弹窗
  showSwitchModal.value = false;
  
  // 跳转到登录页面添加新账户
  router.push('/login');
};
</script>

<style scoped>
.profile-center {
  padding: 0 20px;
  max-width: 1200px;
  margin: 0 auto;
}

.error-message {
  background-color: #fee;
  border: 1px solid #fcc;
  color: #c00;
  padding: 12px;
  border-radius: 6px;
  margin-bottom: 15px;
  font-size: 0.9rem;
  text-align: center;
}

.header-section {
  padding: 25px;
  margin-bottom: 20px;
  text-align: center;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.avatar-section {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.avatar {
  width: 100px;
  height: 100px;
  border-radius: 50%;
  object-fit: cover;
  margin-bottom: 15px;
  border: 4px solid #ddd;
}

.user-info {
  text-align: center;
}

.username {
  margin: 0;
  color: #2c3e50;
  font-size: 1.8rem;
}

.user-role {
  color: #7f8c8d;
  margin: 5px 0 0 0;
}

.settings-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(350px, 1fr));
  gap: 20px;
  margin-bottom: 20px;
}

.setting-category {
  padding: 25px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.category-header h3 {
  color: #2c3e50;
  margin: 0 0 10px 0;
  font-size: 1.3rem;
}

.category-header p {
  color: #7f8c8d;
  margin: 0;
}

.category-actions {
  display: flex;
  gap: 10px;
  margin-top: 20px;
  justify-content: flex-end;
}

.system-info {
  padding: 25px;
  margin-bottom: 20px;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 20px;
  margin-top: 15px;
}

.info-item {
  display: flex;
  flex-direction: column;
}

.info-item label {
  font-weight: 600;
  color: #2c3e50;
  margin-bottom: 5px;
  font-size: 0.9rem;
}

.info-item p {
  color: #7f8c8d;
  margin: 0;
}

.system-actions {
  padding: 25px;
}

.actions-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 15px;
  margin-top: 15px;
}

.action-btn {
  padding: 20px;
  border-radius: 12px;
  border: none;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 10px;
  font-size: 1rem;
  transition: all 0.3s ease;
  text-align: center;
}

.action-btn:hover {
  transform: translateY(-3px);
  box-shadow: 0 10px 20px rgba(0, 0, 0, 0.1);
}

.action-btn i {
  font-size: 1.8rem;
}

.action-btn.danger {
  background: linear-gradient(135deg, #ff416c 0%, #ff4b2b 100%);
  color: white;
}

.action-btn.warning {
  background: linear-gradient(135deg, #ffb347 0%, #ffcc33 100%);
  color: white;
}

.action-btn.primary {
  background: linear-gradient(135deg, #6a11cb 0%, #2575fc 100%);
  color: white;
}

.action-btn.secondary {
  background: linear-gradient(135deg, #6c757d 0%, #495057 100%);
  color: white;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  padding: 25px;
  border-radius: 12px;
  width: 90%;
  max-width: 500px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
}

.modal-content h3 {
  margin-top: 0;
  color: #e74c3c;
  text-align: center;
}

.input-group {
  margin-bottom: 15px;
}

.input-group label {
  display: block;
  margin-bottom: 5px;
  color: #2c3e50;
  font-weight: 600;
  font-size: 0.9rem;
}

.input-group input {
  width: 100%;
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 1rem;
  box-sizing: border-box;
}

.input-group input:focus {
  outline: none;
  border-color: #e74c3c;
  box-shadow: 0 0 0 2px rgba(231, 76, 60, 0.1);
}

.modal-actions {
  display: flex;
  gap: 15px;
  justify-content: center;
  margin-top: 20px;
}

.btn {
  padding: 10px 20px;
  border-radius: 6px;
  border: none;
  cursor: pointer;
  font-size: 1rem;
}

.btn-secondary {
  background-color: #6c757d;
  color: white;
}

.btn-danger {
  background-color: #dc3545;
  color: white;
}

@media (max-width: 768px) {
  .settings-grid {
    grid-template-columns: 1fr;
  }
  
  .actions-grid {
    grid-template-columns: 1fr;
  }
  
  .category-actions {
    flex-direction: column;
  }
  
  .info-grid {
    grid-template-columns: 1fr;
  }
  
  .modal-content {
    width: 95%;
    margin: 0 20px;
  }
  
  .modal-actions {
    flex-direction: column;
  }
}
</style>