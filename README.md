# Study 2023/9/16
在跟随李炎恢老师的thinkphp6微实战视频

https://www.bilibili.com/video/BV1fz411q7km?p=1&vd_source=eb259514a7bfe28fd704b2be0dddf1a4

进行学习的时候发现并解决的一些问题：

  1.对Auth模型中进行多对多关联模型的数据更新和删除;
  
  2.在Auth模块的edit修改页内通过{volist}遍历出Role表的权限复选框的默认选项;

  3.于一对一关联模型User模块的Hobby模型与User模型存在mysql数据表的有关于外键的参数设置问题;
  
  4.在Auth模块的delete方法进行多对多关联模型的数据删除;
  
  5.字段验证问题;

<h2>1.多对多关联模型的数据更新和删除</h2>


```
  public function update(Request $request, $id)
      {
          $data = $request->param();
          try {
              validate(AuthValidate::class)
                  ->scene('edit')
                  ->batch(true)
                  ->check($data);
          }catch(ValidateException $exception){
              return view($this->toast,[
                  'infos'  =>  $exception->getError(),
                  'url_text'  =>  '返回上一页',
                  'url_path'  =>  url('/auth/'.$id.'/edit')
              ]);
          }

  //      -----以上为验证器验证传参内容，往下则准备写入数据库-----
          $user = Authmodel::find($id);
  //        判断密码是否存在 true则写入
          if (!empty($data['NewPasswordAga'])){
              $data['password'] = sha1($data['NewPasswordAga']);
  //          直接将加密后的密码存入Auth表
              $user->save($data);
          }
  
  //        定义多对多关联更新变量：
          $userSaveRole = $user->role()->sync($data['role']);
          return $userSaveRole ? view($this->toast,[
              'infos'     =>  ['修改成功！'],
              'url_text'  =>  '返回首页',
              'url_path'  =>  url('/auth')
          ]) : '修改失败！';
      }
```

<h2>2.解决{volist}遍历出Role表的权限复选框的默认选项问题</h2>

<p>首先通过Auth模块的edit方法内定义要传参的变量：

当时被这个问题卡了一整天，一直在重复捣鼓如何将$data['role']下的一个['type']导入到一个数组中。

但是只要我return $data['role']['type']时，一直返回一个错误：未定义数组type。</p>

```
      public function edit($id)
      {
          $list   =   RoleModel::select();
          $obj    =   Authmodel::with('role')->find($id);
          $ObjRole = $obj->role;
          $roleIds = [];
          foreach ($ObjRole as $type){
              $roleIds[] =  $type->id;
          }
  
          $listWhereIn = RoleModel::whereIn('id',$roleIds)->select();
          $end = [];
          foreach ($listWhereIn as $item) {
              $end[] =  $item->type;
          }
  
          return view('edit',[
              'obj'   =>  $obj,
              'list'  =>  $list,
              'end'   =>  $end
          ]);
      }
```
<p>此时已定义完成所需要的参数并传递至视图:</p>


```
  {volist name="list" id='set'}
  <label for="{$set.id}" class="col-form-label">
      <input type="checkbox" id="{$set.id}" name="role[]" value="{$set.id}"
//    --------------------------------------------
      {if in_array($set.type, $end)}checked{/if}>
//    --------------------------------------------
      <span class="col-form-label-sm">{$set.type}</span>
  </label>
  {/volist}
```

<h2>3.Mysql数据表对外键约束的参数设置问题;</h2>
Mysql外键约束的四个参数：

1.CASCADE 主表中删除或更新记录时，自动删除或更新从表中对应的记录；

2.SET NULL  主表中删除或更新记录时，使用NULL替换从表中对应的记录（不适用于已标记为NOT NULL的字段）；

3.NO ACTION  拒绝主表删除或修改外键关联列；

4.RESTRICT 拒绝主表删除或修改外键关联列（在未定义ON DELETE AND ON UPDATE子句时，这是默认设置）；

<p>这是纯纯Mysql基础不行的表现，起因时当时我创建表的时候，按照的是当初学习PHP时候的命令模板，导致设置的参数为RESTRICT，所以我无法删除任何关联数据。

所以当我将参数设置更改为CASCADE后，问题解决。↓这是我在创建多对多关联模型的数据表：可作为往后的参照。</p>

```
create table `blog_auth`(
    `id` mediumint(8) not null auto_increment primary key,
    `name` varchar(20) not null,
    `password` char(40) not null
)engine=innodb;
create table `blog_role`(
    `id` mediumint(8) not null auto_increment primary key,
    `type` varchar(20) not null
)engine=innodb;
create table `blog_access`(
    `id` mediumint(8) not null auto_increment primary key,
    `auth_id` mediumint(8) not null,
    `role_id` mediumint(8) not null,
    foreign key(auth_id) references blog_auth(id)
        on update cascade on delete cascade,
    foreign key(role_id) references blog_role(id)
        on update cascade on delete cascade
)engine=innodb;
```

<h2>4.多对多关联模型的数据删除</h2>
<p>其实是跟一对一关联删除是一样的，这个在视频中有讲解。只要删除Auth中的参数，与之相关联的中间表由于外键约束的关系而一并删除。</p>

```
public function delete($id)
{
    $setModel = Authmodel::with('role')->find($id);
    return $setModel->delete() ? view($this->toast,[
        'infos'     =>  ['删除成功！'],
        'url_text'  =>  '返回首页',
        'url_path'  =>  url('/auth')
    ]) : '删除失败！';
}
```


