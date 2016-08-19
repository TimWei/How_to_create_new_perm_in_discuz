#Tim's Notepad#

### 介紹 ###
Discuz!起於2001年，</br>
是中國一套開放但不開源的論壇框架，以PHP語言寫成</br>
該論壇框架包含了許多常見的功能與插件之外</br>
有針對中國境內常用的社群網路(QQ、人人、微信)做整合</br>
可以快速的部屬功能強大的論壇網站</br>


### 教學目的 ###
雖然在Discuz!已經內建許多強大的功能，不過偶還是會需要自定義後台功能</br>
此篇將逐步建置一個自訂義的功能在Discuz!後台中

### 檔案目錄說明 ###
1. source\language\\.
	該目錄內的檔案為字串的HashTable,可用於實現I18n多語系功能
2. source\class\\.
	放了許多的框架預設的class/helpers/configs。 一些三方整合的設定檔也於該目錄內
3. source\admincp\\.
	該目錄內檔案為後台各operation的方法實現與html_templating。 檔案的規則為 "admincp_" + action_name

### 實做 ###

以下將以"在用戶管理(action=members)內新增一個導出Table資料的功能"來做介紹

1. 在source\language\lang_admincp.php的Hashtable中新增字串的映射
	```			
		$lang = array
		(
	+		// tim's custom
	+		'tim_nav_table_export' => '導出',	
	+		'tim_export_tip' => '導出數據',
	+		'tim_export_select_name' => '導出Table',
	+		'tim_export_post_invalid' => '表單資料有誤',
	+		'tim_founder_perm_members_tim' => '匯出(查看)',
	+		'tim_founder_perm_members_tim_post' => '匯出(執行)',
	```

2. 首先我們要在用戶管理的子頁面頁籤新增我們的操作頁面連結，首先找到source\admincp\admincp_members.php中判斷operation的if結構式，用戶管理的頁籤原本為'searcy','clean','repeat'，在此我們要在原三個operation中皆加入頁籤。
	```
		if($operation == 'search') {

			if(!submitcheck('submit', 1)) {

				shownav('user', 'nav_members');
				showsubmenu('nav_members', array(
					array('search', 'members&operation=search', 1),
					array('nav_repeat', 'members&operation=repeat', 0),
	+				array('tim_nav_table_export', 'members&operation=tim', 0),
				));
	```

	```
		} elseif($operation == 'repeat') {

			if(empty($_GET['uid']) && empty($_GET['username']) && empty($_GET['ip'])) {

				shownav('user', 'nav_members');
				showsubmenu('nav_members', array(
					array('search', 'members&operation=search', 0),
					array('clean', 'members&operation=clean', 0),
					array('nav_repeat', 'members&operation=repeat', 1),
	+				array('tim_nav_table_export', 'members&operation=tim', 0),
				));
	```

	```
		} elseif($operation == 'clean') {

			if(!submitcheck('submit', 1) && !submitcheck('deletesubmit', 1)) {

				shownav('user', 'nav_members');
				showsubmenu('nav_members', array(
					array('search', 'members&operation=search', 0),
					array('clean', 'members&operation=clean', 1),
					array('nav_repeat', 'members&operation=repeat', 0),
	+				array('tim_nav_table_export', 'members&operation=tim', 0),
				));
	```
3. 在判斷operation的if結構式中新增新operation的執行區塊

	```
	+			} elseif($operation == 'tim'){

	+			} elseif($operation == 'tim_post') {

			} elseif($operation == 'export') {
				$uids = searchmembers($search_condition, 10000);
				$detail = '';
				if($uids && is_array($uids)) {
	```

4. 由於opertation=tim_post是匯出csv。不需要相關的html header tag，在檔案片頭的判斷式加入這個判斷

	```
	+	if($operation != 'export' && $operation != 'tim_post') {
	-	if($operation != 'export' ) {
	```

5. 在source\class\class_core.php中新增一Hashtable做為表單的Table白名單，之後要修改可選的table從這修改即可

	```
			define('IN_DISCUZ', true);
			define('DISCUZ_ROOT', substr(dirname(__FILE__), 0, -12));
			define('DISCUZ_CORE_DEBUG', false);
			define('DISCUZ_TABLE_EXTENDABLE', false);
	+		define('TIM_SELECT_HASHTABLE',  serialize ( array( 't1' => 'pre_common_member', 't2' => 'pre_common',)));

			set_exception_handler(array('core', 'handleException'));
	```

6. 回到source\admincp\admincp_members.php中，在判斷operation的if結構式中實做operation=tim的程式區塊

	```
		} elseif($operation == 'tim'){
	+			// navbar and submenu
	+			shownav('user', 'nav_members');
	+			showsubmenu('nav_members', array(
	+				array('search', 'members&operation=search', 0),
	+				array('clean', 'members&operation=clean', 0),
	+				array('nav_repeat', 'members&operation=repeat', 0),
	+				array('tim_nav_table_export', 'members&operation=tim', 1),
	+			));
	+			// set tips
	+			showtableheader('tim_export_tip', '', 'id=tips' , 0);
	+			showtablefooter();
	+
	+			// form's helper
	+			$tables = unserialize(TIM_SELECT_HASHTABLE);
	+			showformheader('members&operation=tim_post');
	+			$selects = array();
	+			foreach($tables as $table_index => $table_name){
	+				array_push($selects, array($table_index,$table_name));
	+			}
	+			showsetting('tim_export_select_name', array('table',$selects), 't1','select');
	+			showsubmit('submit', 'submit');
	+			showformfooter();

	```
7. 在判斷operation的if結構式中實做operation=tim_post的程式區塊

	```
		} elseif($operation == 'tim_post') {
	+		if(submitcheck('submit')) {
	+			//get selected table
	+			$tables = unserialize(TIM_SELECT_HASHTABLE);
	+			$selected = $tables[ $_POST['table'] ];
	+				//get primary key
	+			$pk = '';
	+			$pk_row = DB::fetch_first("SHOW KEYS FROM ".$selected." WHERE Key_name = 'PRIMARY'");	
	+			if( !empty($pk_row) ){
	+				$pk = $pk_row['Column_name'];
	+			}
	+
	+			//get column names
	+			$all_col = DB::fetch_all("SHOW COLUMNS FROM ".$selected,null,'Field');
	+
	+			// get all column name
	+			$all_attr = '';
	+			foreach($all_col as $key => $value){
	+				$all_attr .= $key.',';
	+			}
	+			$all_attr = substr($all_attr, 0, -1)."\n";
	+
	+			//csv headers
	+			$result = $all_attr;
	+
	+			// append rows into string
	+			$data = DB::fetch_all("SELECT * FROM ".$selected, null, $pk);
	+			foreach($data as $row){
	+				foreach($row as $attr){
	+					$result .= $attr.',';
	+				}
	+				$result .= "\n";
	+			}
	+			$filename = $selected.'_'.date('Ymd', TIMESTAMP).'.csv';
	+			ob_end_clean();
	+			header('Content-Encoding: none');
	+			header('Content-Type: application/octet-stream');
	+			header('Content-Disposition: attachment; filename='.$filename);
	+			header('Pragma: no-cache');
	+			header('Expires: 0');
	+			echo $result;
	+			exit();
	+		}else{
	+			cpheader();
	+			cpmsg('tim_export_post_invalid', '', 'error');
	+		}
	```
8. 接下來要新增新權限的master data, 要注意的是該master data不是以數據的形式保存，而式hard code在source\admincp\admin_cp_perm.php中。 

	```
		array_splice($menu['user'], 1, 0, array(
			array('founder_perm_members_group', 'members_group'),
			array('founder_perm_members_access', 'members_access'),
			array('founder_perm_members_credit', 'members_credit'),
			array('founder_perm_members_medal', 'members_medal'),
			array('founder_perm_members_repeat', 'members_repeat'),
			array('founder_perm_members_clean', 'members_clean'),
			array('founder_perm_members_edit', 'members_edit'),
	+		array('tim_founder_perm_members_tim', 'members_tim'),
	+		array('tim_founder_perm_members_tim_post', 'members_tim_post'),
		));
	```
9. 你成功新創一個權限與功能了!