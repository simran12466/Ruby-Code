# Ruby-Code
Ruby on rail

source 'https://rubygems.org'
gem 'rails', '~> 7.0'
gem 'pg'
gem 'devise'
gem 'pundit'
gem 'bootstrap', '~> 5.3'
gem 'jquery-rails'

group :development, :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
end

# Database Migration for Users
def change
  create_table :users do |t|
    t.string :name, null: false, limit: 60
    t.string :email, null: false, unique: true
    t.string :encrypted_password, null: false
    t.string :address, limit: 400
    t.integer :role, default: 1, null: false # 0 = Admin, 1 = Normal User, 2 = Store Owner

    t.timestamps
  end
end

# Database Migration for Stores
def change
  create_table :stores do |t|
    t.string :name, null: false, limit: 60
    t.string :email
    t.string :address, null: false, limit: 400
    t.float :average_rating, default: 0.0

    t.timestamps
  end
end

# Database Migration for Ratings
def change
  create_table :ratings do |t|
    t.references :user, null: false, foreign_key: true
    t.references :store, null: false, foreign_key: true
    t.integer :rating, null: false

    t.timestamps
  end
end

# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  enum role: { admin: 0, normal_user: 1, store_owner: 2 }

  validates :name, presence: true, length: { minimum: 20, maximum: 60 }
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }, uniqueness: true
  validates :address, length: { maximum: 400 }
  validates :password, presence: true, length: { in: 8..16 }, format: { with: /(?=.*[A-Z])(?=.*[!@#$&*])/ }

  has_many :ratings, dependent: :destroy
end

# app/models/store.rb
class Store < ApplicationRecord
  validates :name, presence: true, length: { maximum: 60 }
  validates :address, presence: true, length: { maximum: 400 }

  has_many :ratings, dependent: :destroy
end

# app/models/rating.rb
class Rating < ApplicationRecord
  belongs_to :user
  belongs_to :store

  validates :rating, presence: true, inclusion: { in: 1..5 }
end

# app/controllers/admin_controller.rb
class AdminController < ApplicationController
  before_action :authenticate_user!
  before_action :authorize_admin

  def dashboard
    @total_users = User.count
    @total_stores = Store.count
    @total_ratings = Rating.count
  end

  def manage_users
    @users = User.all
  end

  def manage_stores
    @stores = Store.all
  end

  private

  def authorize_admin
    redirect_to root_path, alert: 'Access Denied' unless current_user.admin?
  end
end


class UsersController < ApplicationController
  def update_password
    if current_user.update(user_params)
      redirect_to root_path, notice: 'Password updated successfully.'
    else
      render :edit
    end
  end

  private

  def user_params
    params.require(:user).permit(:password, :password_confirmation)
  end
end

# app/controllers/stores_controller.rb
class StoresController < ApplicationController
  before_action :authenticate_user!

  def index
    @stores = Store.all
  end

  def rate
    @store = Store.find(params[:id])
    @rating = @store.ratings.find_or_initialize_by(user: current_user)
    @rating.update(rating_params)

    redirect_to stores_path, notice: 'Rating submitted.'
  end

  private

  def rating_params
    params.require(:rating).permit(:rating)
  end
end

# app/views/admin/dashboard.html.erb
<h1>Admin Dashboard</h1>
<p>Total Users: <%= @total_users %></p>
<p>Total Stores: <%= @total_stores %></p>
<p>Total Ratings Submitted: <%= @total_ratings %></p>

# app/views/stores/index.html.erb
<h1>Store List</h1>
<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Address</th>
      <th>Average Rating</th>
      <th>My Rating</th>
      <th>Actions</th>
    </tr>
  </thead>
  <tbody>
    <% @stores.each do |store| %>
      <tr>
        <td><%= store.name %></td>
        <td><%= store.address %></td>
        <td><%= store.average_rating %></td>
        <td><%= current_user.ratings.find_by(store: store)&.rating || 'N/A' %></td>
        <td>
          <%= form_with model: [store, current_user.ratings.find_or_initialize_by(store: store)], url: rate_store_path(store), method: :post do |form| %>
            <%= form.number_field :rating, in: 1..5 %>
            <%= form.submit 'Submit' %>
          <% end %>
        </td>
      </tr>
    <% end %>
  </tbody>
</table>
      
class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user = user
    @record = record
  end

  def admin?
    user.admin?
  end
end
