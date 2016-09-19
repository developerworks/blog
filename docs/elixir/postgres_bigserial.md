> ### 要解决的问题
> 数据库中存储的行数超过了 `serial` 类型所能容纳的数量, 因此需要采用 `bigserial` 类型的整数作为主键
> `serial` 的取值范围为: `1 到 2147483647`
> `bigserial` 的取值范围为: `1 到 9223372036854775807`

完整的 Postgresql 字段的数据类型, 可以参考[这里](https://www.postgresql.org/docs/9.1/static/datatype-numeric.html)

![图片描述][1]

### 迁移脚本

```elixir
defmodule ElectricProto.Repo.Migrations.AddStationTable do
  use Ecto.Migration

  def up do
    create table(:station, primary_key: false) do
      add :id,          :bigserial, primary_key: true
      timestamps
    end
  end

  def down do
    drop table(:station)
  end
end
```

> 要点
> 1. `create table`的参数`primary_key`要设置为`false`, 
> 2. 通过`add`宏指定主键列`id`, 类型为`bigserial`

### 模型的声明

```elixir
@primary_key {:id, :id, autogenerate: true}

schema "station" do
  field :area,        :string,  default: ""
  field :carrier,     :string,  default: ""
  field :city,        :string,  default: ""
  field :deployed,    :boolean, default: false
  field :description, :string,  default: ""
  field :device_auth, :string,  default: ""
  field :device_type, :string,  default: ""
  field :geolocation, :string,  default: ""
  field :ip_addr,     :string,  default: ""
  field :qrcode,      :string,  default: ""
  field :station_id,  :string,  default: ""
  field :status,      :string,  default: ""
  timestamps
end
```

> 要点
> 1. 主键要声明为`:id`类型, `@primary_key {:id, :id, autogenerate: true}`

完!
  [1]: https://segmentfault.com/img/bVyKb4
