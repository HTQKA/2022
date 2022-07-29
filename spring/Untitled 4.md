```
	

	public void syncEHRUSerInfo() {
		SyncCommonService.me.syncEHRUserInfo();
		forwardAction("/sync/syncCommon");
	}

```



```
	
	/**
	 * <sql id="mssql-gfzEhrData"><![CDATA[
	With CA As (Select orgid, orgcode, orgname, Cast(orgname As Varchar(Max)) As fullpath From view_helios_orglist {$orgcodes#Where orgcode IN (@INSTR)$}
	Union All Select A.orgid, A.orgcode, A.orgname, Cast(B.fullpath + '\' + A.orgname As Varchar(Max)) As fullpath From view_helios_orglist A Join CA B On A.sub_orgcode = B.orgcode)
	Select A.*, B.fullpath From view_hrm_gfz_list A Join CA B On A.deptcode = B.orgcode
	]]></sql>
	 */
	//
	public  void syncEHRUserInfo(){
		log.info("开始执行EHR同步任务");
		String resValue = CtSysResourceService.me.getResValue("MES_EHR_DEPT_MAPPING", "MES_EHR_DEPT_MAPPING");
		JSONObject jsonObject = JSONObject.parseObject(resValue);
		String orgcodes = jsonObject.getString("15088005");
		List<Record> listRecord = listRecord("mssql_ehr","sync.mssql-gfzEhrData",IUtils.createLinkedCaseInsensitiveMap("orgcodes",orgcodes));
		List<Record> compareUser = compareUser(listRecord);
		for(Record record : compareUser){
			log.info(record.toString());
		}
		insertUserData(compareUser);
		//对比user表,
		//EHR有的用户,user表没有,user表新增,user_info,work,user_role(用户基本角色),ct_aps_team_user新增一条数据
		//EHR有的用户,user离职了(应该不存在该情况)
		//EHR有但是离职了,ct_aps_team_user离职状态改为1
		
		
	}
	/**
	 * 比较EHR和MES数据()
	 * 1. mes不存在lastname 新增 (查看 work_id 是否存在 ,存在查看)
	 * 2. mes存在lastname 
	 * 	  2.1 而且work_id一致,更新 
	 * 	  2.2 work_id 不一致,
	 * 		2.2.1查询是否有
	 * @param list
	 * @return
	 */
	
	//湖州屏蔽料唯一存在一个EHR workcode 对应多个 mes work_id 用户为:罗亦斌 ,11723
	//对比user表数据,筛选EHR有的用户,user表没有的
	public List<Record> compareUser(List<Record> list){
		//查用user高分子所有用户
		//List<Record> listRecord = listRecord("Select id, title, user_name,work_id,status From ct_sys_user A Where 1 = 1 And Exists (Select 1 From ct_sys_user_work AA Where Left(AA.dept_code, 4) In ('1008', '1108', '1308', '1508', '1208', '3008') And A.id = AA.user_id)");
		//查询所有用户进行对比
		List<Record> listRecord = listRecord("select * from ct_sys_user");

		//新增用户
		List<Record> listNewRecord = new ArrayList<>();
		//新增加后缀
		List<Record> listNewRecord1 = new ArrayList<>();
		//user表更新用户
		List<Record> listUserRecord = new ArrayList<>();
		//teamuser表更新
		List<Record> listTeamUserRecord = new ArrayList<>();
		//不操作用户
		List<Record> noActionRecord = new ArrayList<>();
		//用户名和状态生成map
		Map<String, String> mapUser = listRecord.stream().collect(Collectors.toMap(e -> e.get("title"), l -> l.get("status")));
		Map<String, String> mapWork = listRecord.stream().collect(Collectors.toMap(k -> k.get("title"), v -> v.get("work_id") == null ? "": v.get("work_id")));
		//对这代
		Map<String, List<String>> workIdMap = workIdMap(listRecord);
		log.info("Map = " + workIdMap.toString());
		//取出所有需要新增的用户
		for(Record record : list){
			String title = IUtils.toSpell(record.get("lastname"));
			if(IStr.isBlank(mapUser.get(title))){
				record.set("title", title);
				listNewRecord.add(record );
				//list.remove(record);
			}else{//mes存在的员工
				//根据工号判断两个员工是否为同一个人
				if(record.get("workcode").equals(mapWork.get(title))){
					//0在职,1离职
					String status = "0".equals(record.get("isleave").toString()) ? "Y" : "N";
					//都在职
					if("Y".equals(mapUser.get(title)) && "0".equals(record.get("isleave").toString())){//都在职
						//不操作
						noActionRecord.add(record);
					}else if("Y".endsWith(mapUser.get(title)) && "1".equals(record.get("isleave").toString())){//Mes在职,EHR不在职,更新team_user的is_Leave状态为"1"
						listTeamUserRecord.add(record);
					}else if("N".equals(mapUser.get(title)) && "0".equals(record.get("isleave").toString())){//Mes不在职,EHR在职 ,更新ct_sys_user表status状态为"Y"
						listUserRecord.add(record);
					}else if("N".equals(mapUser.get(title)) && "1".equals(record.get("isleave").toString())){//Mes不在职,EHR不在职
						//不操作
						noActionRecord.add(record);
					}
				}else{
					//不是同一个人
					//根据title查询user是否有其他名称一样的账号
					//List<Record> listRecord2 = listRecord("select * from ct_sys_user where title like ? and title != ?",title,title);
					List<String> string = workIdMap.get(record.get("workcode"));
					if(string == null || string.size() == 0){//mes中work_id为空
						//需要新增 ,且title 后要加数字
						log.info("需要新增");
						int sameName = sameName(listRecord, title);
						record.set("title",title+sameName);
						listNewRecord.add(record);
						listNewRecord1.add(record);
						//list.remove(record);
						
					}else{
						//工号存在,查找对应工号有多少个在几个账户,
						log.info("work_id =  {} ,list = {}",record.get("workcode"),string  );
						if(string.size() == 1){
							//工号对应的title只有一个,对应EHR用户,判断在职状态是否一致
							String status = "0".equals(record.get("isleave").toString()) ? "Y" : "N" ;
							//都在职
							if("Y".equals(mapUser.get(title)) && "0".equals(record.get("isleave").toString())){
								//不操作
								noActionRecord.add(record);
							}else if("Y".endsWith(mapUser.get(title)) && "1".equals(record.get("isleave").toString())){//Mes在职,EHR不在职,更新team_user的is_Leave状态为"1"
								listTeamUserRecord.add(record);
							}else if("N".equals(mapUser.get(title)) && "0".equals(record.get("isleave").toString())){//Mes不在职,EHR在职 ,更新ct_sys_user表status状态为"Y"
								listUserRecord.add(record);
							}else if("N".equals(mapUser.get(title)) && "1".equals(record.get("isleave").toString())){//Mes不在职,EHR不在职
								//不操作
								noActionRecord.add(record);
							}
						}else{
							//对应多个并且有一个是在职状态的
							for(String a : string){
								String status = mapUser.get(a);
								if("Y".equals(status)){
									//匹配ERH账号,
									record.set("title", a);
									break;
								}
							}
						}
					}
					//log.info("你好 = " + string);
				}
				log.info("用户:{} ,状态:{}" ,title,mapUser.get(title));
			}
		}
		//String collect1 = listRecord3.stream().map(e -> String.format("'%s'", e.getStr("user_name"))).distinct().collect(Collectors.joining(","));

		String newUser = listNewRecord.stream().map(e -> String.format("'%s'",IUtils.toSpell(e.getStr("lastname")))).distinct().collect(Collectors.joining(","));
		String noUSer = noActionRecord.stream().map(e -> String.format("'%s'", IUtils.toSpell(e.getStr("lastname")))).distinct().collect(Collectors.joining(","));
		String teamUser = listTeamUserRecord.stream().map(e -> String.format("'%s'",IUtils.toSpell(e.getStr("lastname")))).distinct().collect(Collectors.joining(","));
		String update = listUserRecord.stream().map(e -> String.format("'%s'", IUtils.toSpell(e.getStr("lastname")))).distinct().collect(Collectors.joining(","));
		String newUser1 = listNewRecord1.stream().map(e -> String.format("'%s'",e.getStr("title"))).distinct().collect(Collectors.joining(","));
		log.info("新增加后缀:" + newUser1);
		log.info("新增:" + newUser);
		log.info("更新:" + update);
		log.info("team更新:" + teamUser);
		log.info("不操作:" + noUSer);
		insertUserData(listNewRecord);
		updateTeamUserDate(listTeamUserRecord);
		updateUserDate(listUserRecord);
		return listNewRecord;
	}
	
	public Map<String, List<String>> workIdMap(List<Record> listRecord){
		Map<String, List<String>> workID = new HashMap<>();
		for(Record record : listRecord){
			String work_id = record.getStr("work_id");
			if(IStr.isNoneBlank(work_id)){
				List<String> list = new ArrayList<>();
				list.add(record.getStr("title"));
				if(workID.containsKey(work_id)){
					list.addAll(workID.get(work_id));
					workID.put(work_id, list);
				}else{
					workID.put(work_id, list);
				}
			}
		}
		return workID;
	}
	
	//存在同名用户,且需要新增,返回需要更新的后缀
	public int sameName(List<Record> list,String title){
		int suffix = 0;
		for(Record record : list){
			if(record.getStr("title").startsWith(title) && ((record.getStr("title").length() == (title.length() + 1)) || (record.getStr("title").length() == (title.length())))){
				suffix++;
			}
		}
		return suffix;
	}
	//user表更新
	public void updateUserDate(List<Record> list){
		if(list != null && list.size() > 0){
			String user_name = list.stream().map(e -> String.format("'%s'", IUtils.toSpell(e.getStr("lastname")))).distinct().collect(Collectors.joining(","));
			String updateSql = String.format("update ct_sys_user set status = 'Y' where title in (%s)", user_name);
			log.info("更新User_sql = {}",updateSql );
			//int i = updateSQL(updateSql);
		} 
	}
	
	//teamUser表更新
	public void updateTeamUserDate(List<Record> list){
		if(list != null && list.size() > 0){
			String team_user = list.stream().map(e -> String.format("'%s'", IUtils.toSpell(e.getStr("lastname")))).distinct().collect(Collectors.joining(","));
			String updateSql = String.format("update ct_aps_team_user set is_leave = '1' where team_user in (%s)" , team_user);
			log.info("更新team_sql = {}" ,updateSql);
			//int i = updateSQL(updateSql);
		}
	}
	
	public void insertUserData(List<Record> list){
		if(list != null && list.size() > 0){
			List<Record> listUser = new ArrayList<>();	
			List<Record> listUserInfo = new ArrayList<>();
			List<Record> listUserWork = new ArrayList<>();
			List<Record> listUserRole = new ArrayList<>();
			List<Record> listTeamUser = new ArrayList<>();
			String resValue = CtSysResourceService.me.getResValue("MES_EHR_DEPT_MAPPING", "MES_EHR_DEPT_MAPPING");
			JSONObject deptJson = JSONObject.parseObject(resValue);
			for(Record record : list){
				//listUser 用户表信息,id,title,pwd,user_name,add_time,status,lm_user,lm_timer,pwd_verify,pwd_expire,work_id,sms_verify
				Record recordUser = new Record();
				long user_id = IUtils.getNextID();
				recordUser.set("id", user_id);
				recordUser.set("title", record.get("title"));
				recordUser.set("pwd", IHash.md5("123456", record.getStr("title")));
				recordUser.set("user_name", record.get("lastname"));
				recordUser.set("mobile_verify", 0);
				recordUser.set("email_verify", 0);
				recordUser.set("add_time", IDate.getTimestamp());
				String status = "0" .equals(record.get("isleave").toString()) ? "Y" : "N";
				recordUser.set("status", status);
				recordUser.set("lm_time", IDate.getTimestamp());
				recordUser.set("pwd_verify", "Y");
				recordUser.set("pwd_expire", new Timestamp(4102415999999l));
				recordUser.set("skey_verify", "N");
				recordUser.set("weixin_verify",	"N");
				recordUser.set("work_id", record.get("workcode"));
				recordUser.set("sms_verify", "N");
				listUser.add(recordUser);
				//==============================================================
				//listUserInfo user_info表,id,user_id,sync_status,
				Record recordUserInfo = new Record();
				long user_info = IUtils.getNextID(); 
				recordUserInfo.set("id", user_info);
				recordUserInfo.set("user_id", user_id);
				recordUserInfo.set("sync_status", "N");
				listUserInfo.add(recordUserInfo);
				//==============================================================
				//listUserWork ,user_work表,id,user_id,dept_code,user_postion(100001),dedault_flag.lm_user,lm_time
				Record recordUserWork = new Record();
				Long work_id = IUtils.getNextID();
				recordUserWork.set("id", work_id);
				recordUserWork.set("user_id", user_id);
				recordUserWork.set("dept_code", deptJson.get(record.getStr("deptcode")));
				recordUserWork.set("user_postion", 100001);// 职务:员工
				recordUserWork.set("dedault_flag", 1);//默认工作
				recordUserWork.set("lm_time", IDate.getTimestamp());
				listUserWork.add(recordUserWork);
				//==============================================================
				//listUserRole ,user_role表,id,work_id,role_id,lm_user,lm_time,
				Record recordUserRole = new Record();
				Long user_role_id = IUtils.getNextID();
				recordUserRole.set("id", user_role_id);
				recordUserWork.set("work_id", work_id);
				recordUserRole.set("role_id", 21);//用户基本角色
				recordUserRole.set("lm_time", IDate.getTimestamp());
				recordUserRole.set("lm_user", "");
				listUserRole.add(recordUserRole);
				//==============================================================
				Record recordTeamUser  = new Record();
				Long team_user_id = IUtils.getNextID();
				recordTeamUser.set("id", team_user_id);
				recordTeamUser.set("team_user", record.get("title"));
				recordTeamUser.set("lm_time", IDate.getTimestamp());
				String is_welfare = "0".equals(record.get("field22").toString()) ? "Y" : "N";
				recordTeamUser.set("is_welfare", is_welfare);
				recordTeamUser.set("is_leave", record.get("isleave"));
				listTeamUser.add(recordTeamUser);
			}
			log.info("CtSysUser = {}" ,listUser );
			log.info("CtSysUserInfo = {}" ,listUserInfo);
			//打印id
			String userId = listUser.stream().map(e -> e.getLong("id").toString()).distinct().collect(Collectors.joining(","));		
			String userInfoId =  listUserInfo.stream().map(e -> e.getLong("id").toString()).distinct().collect(Collectors.joining(","));
			String userWorkId = listUserWork.stream().map(e -> e.getLong("id").toString()).distinct().collect(Collectors.joining(","));
			String userRoleid = listUserRole.stream().map(e -> e.getLong("id").toString()).distinct().collect(Collectors.joining(","));
			String teamUserId = listTeamUser.stream().map(e -> e.getLong("id").toString()).distinct().collect(Collectors.joining(","));
			log.info("userId = {}",userId);
			log.info("userInfoId = {}" ,userInfoId);
			log.info("userWorkId = {}",userWorkId);
			log.info("userRoleid = {}" ,userRoleid);
			log.info("teamUserId = {}",teamUserId);
			//autoBatchInsert(CtSysUser.class,listUser );
			//autoBatchInsert(CtSysUserInfo.class, listUserInfo);
			//autoBatchInsert(CtSysUserWork.class, listUserWork);
			//autoBatchInsert(CtSysUserRole.class,listUserRole );
			//autoBatchInsert(CtApsTeamUser.class, listTeamUser);
		}
	}
	
	//查询MES ct_sys_user表中所有在职但是没有工号的数据
	public void updateWorkId1(){
		String sql = "select * from ct_sys_user where status = 'Y' and COALESCE(work_id,'') = '' ";
		List<Record> listRecord = listRecord(sql);
		List<Record> listRecord3 = listRecord("select user_name ,count(user_name) from ct_sys_user GROUP BY user_name HAVING count(user_name) > 1 ");
		String collect = listRecord.stream().map(e -> String.format("'%s'", e.getStr("user_name"))).distinct().collect(Collectors.joining(","));
		//查询EHR中的work_id
		
		String collect1 = listRecord3.stream().map(e -> String.format("'%s'", e.getStr("user_name"))).distinct().collect(Collectors.joining(","));
		//查询work_id为空的人员
		List<Record> listRecord2 = listRecordAt("mssql_ehr", String.format("select * from view_hrm_gfz_list where lastname in (%s) ", collect));
		//查询是否名字重复
		List<Record> queryListAt = listRecordAt("mssql_ehr", String.format("select * from view_hrm_gfz_list where lastname in (%s) ", collect1));
		Map<String, String> map1 = queryListAt.stream().collect(Collectors.toMap(k-> IUtils.toSpell(k.get("lastname")), v -> v.get("workcode")));
		Map<String, String> map = listRecord2.stream().collect(Collectors.toMap(k -> IUtils.toSpell(k.get("lastname")), v -> v.get("workcode")));
		for(Record record : listRecord){
			record.set("work_id", map.get(record.get("lastname")));
		}
		//autoBatchUpdate(CtSysUser.class, listRecord, "id");
	}
	
	public void updateWorkId(){
		String sql = "Select id, title, user_name From ct_sys_user A Where Coalesce(work_id, '') = '' And Exists (Select 1 From ct_sys_user_work AA Where Left(AA.dept_code, 4) In ('1008', '1108', '1308', '1508', '1208', '3008') And A.id = AA.user_id)";
		//高分子work_id为空员工信息
		List<Record> listUser = listRecord(sql);
		String[] name = {"黄凯","王万良"};
		String names = listUser.stream().map(e-> String.format("'%s'", e.getStr("user_name"))).distinct().collect(Collectors.joining(","));
		
		//查询EHR改部分用户
		String sqlEHR =  String.format("With CA As (Select orgid, orgcode, orgname, Cast(orgname As Varchar(Max)) As fullpath From view_helios_orglist Where orgcode = '9' Union All Select A.orgid, A.orgcode, A.orgname, Cast(B.fullpath + '\' + A.orgname As Varchar(Max)) As fullpath From view_helios_orglist A Join CA B On A.sub_orgcode = B.orgcode)	Select A.*, B.fullpath From view_hrm_gfz_list A Join CA B On A.deptcode = B.orgcode	where A.lastname in (%s)",names);
		List<Record> listUserEHR= listRecordAt("mssql_ehr", sqlEHR);
		
		List<Record> listSpecialUser = new ArrayList<>();
		for(Record record : listUserEHR){
			if("黄凯".equals(record.get("lastname"))){
				listSpecialUser.add(record);
				listUser.remove(record);
			}
		}
		
		//listUserEHR,根据lastname去重,保留work_id 较大的数据
		
		
		Map<String, String> map = listDeduplication(listUserEHR);
		
		for(Record record : listUser){
			if("黄凯".equals(record.get("user_name"))){
				if("huangkai".equals(record.get("title"))){
					record.set("work_id", "17707");
				}else{
					record.set("work_id", "17750");
				}
			}else{
				record.set("work_id", map.get(record.get("user_name")));
			}
			log.info("User = " + record);
		}
		log.info("ListUser = " + listUser.toString());
	}
	
	public Map<String, String> listDeduplication(List<Record> list){
		Map<String, String> map  =  new HashMap<>();
		for(Record record : list){
			String key = record.getStr("lastname");
			String value = record.getStr("workcode");
			if(map.containsKey(key)){
				//保留work_id 较大的数据
				if(Integer.parseInt(map.get(key)) < Integer.parseInt(value)){
					map.replace(key, value);
				}
			}else{
				map.put(key, value);
			}
		}
		return map;
	}

```



```

```



```
update   
	     t1
	     set 
	  t1.AUTOUPDATETIME=NOW()

	 from 
	CLBKCRM_ORDER_HEADER  t1
inner  join CLBKCRM_ORDER t2
on t1.vbeln=t2.vbeln 

where  ADD_SECONDS (TO_TIMESTAMP (t2.AUTOUPDATETIME ), 60*2)>=?
```

```
update   
      t1
      set 
   t1.AUTOUPDATETIME=NOW()
from
CLBKCRM_ORDER_HEADER t1
inner join 
(select t2.vbeln,max(t2.AUTOUPDATETIME) as AUTOUPDATETIME
 from CLBKCRM_ORDER t2
group by t2.vbeln) t2
on t1.vbeln=t2.vbeln
where t1.AUTOUPDATETIME<t2.AUTOUPDATETIME
```



```
update   
	     t1
	     set 
	  t1.AUTOUPDATETIME=NOW()

	 from 
	CLBKCRM_INV_HEADER  t1
inner  join CLBKCRM_INV_ITEM t2
on t1.ZGTFP=t2.ZGTFP

where  ADD_SECONDS (TO_TIMESTAMP (t2.AUTOUPDATETIME ), 60*2)>=?
```

回款

```

update   
      t1
      set 
   t1.AUTOUPDATETIME=NOW()
from
CLBKCRM_INV_HEADER t1
inner join 
(select t2.ZGTFP,max(t2.AUTOUPDATETIME) as AUTOUPDATETIME
 from CLBKCRM_INV_ITEM t2
group by t2.ZGTFP) t2
on t1.ZGTFP=t2.ZGTFP
where t1.AUTOUPDATETIME<t2.AUTOUPDATETIME
```



发货单

```
update   
	     t1
	     set 
	  t1.AUTOUPDATETIME=NOW()

	 from 
	CLBKCRM_DELIVERY_HEADER  t1
inner  join CLBKCRM_DELIVERY_ITEM t2
on t1.vbeln=t2.vbeln 

where  ADD_SECONDS (TO_TIMESTAMP (t2.AUTOUPDATETIME ), 60*2)>=?
```

```
update   
      t1
      set 
   t1.AUTOUPDATETIME=NOW()
from
CLBKCRM_DELIVERY_HEADER t1
inner join 
(select t2.vbeln,max(t2.AUTOUPDATETIME) as AUTOUPDATETIME
 from CLBKCRM_DELIVERY_ITEM t2
group by t2.vbeln) t2
on t1.vbeln=t2.vbeln
where t1.AUTOUPDATETIME<t2.AUTOUPDATETIME
```

