---

name: tsinswreng-avln-dsl

description: Avalonia Avln.Dsl庫 UI寫法規範。開發Avalonia項目時應閱讀此skill

---

## 命名規範

- 視圖(View)用 View前綴
- 視圖模型(ViewModel) 用Vm前綴

## 文件位置

例:

```
MyProj/Views/
	Login/
		ViewLogin.cs
		ViewLogin.Impl.cs
		VmLogin.cs
		VmLogin.Impl.cs
	UserProfile/
		ViewUserProfile.cs
		ViewUserProfile.Impl.cs
		VmUserProfile.cs
		VmUserProfile.Impl.cs
```

- ViewXxx和VmXxx須同時放在名为Xxx的文件夾下
- 遵守 《聲明與實現分離》的規範、類型和所有函數都聲明爲partial、`Xxx.cs`中不寫函數實現、函數實現都寫在`Xxx.Impl.cs`中

## Vm規範

Vm即ViewModel。 示例代碼:

`VmUserProfile.cs`:

```cs
namespace MyProj.Views.UserProfile;
using Ctx = VmUserProfile;
///記得在這裏寫註釋
public partial class VmUserProfile: ViewModelBase, IMk<Ctx>{
	//所有Vm都要有protected的無參構造器。
	protected VmUserProfile(){}
	
	//用于從外部直接創建對象、不注入依賴。
	//不能定義多個public構造器、否則依賴注入會出錯。
	public static Ctx Mk(){
		return new Ctx();
	}
	
	//聲明依賴。不需要加任何修飾符、不需要{get;set;}、都初始化爲=default!。
	ISvcUser SvcUser = default!;
	ISvcUserCtx SvcUserCtx = default!;
	
	//唯一的public構造器、用于依賴注入。
	public partial VmUserProfile(
		ISvcUser? SvcUser;
		ISvcUserCtx? SvcUserCtx;
	);
	
	//用于綁定的屬性的getter和setter必須定義成這樣、無特殊情況(如轉發其他屬性)則必須使用field關鍵字。
	public int Cnt1{
		get;
		set{SetProperty(ref field, value);}
	}=0;
	
	public str Input{
		get;
		set{SetProperty(ref field, value);}
	}="";

	public ObservableCollection<str> List{
		get;
		set{SetProperty(ref field, value);}
	}=["a","b","c"];
	
	///記得在這裏寫註釋
	/// 所有函數(包括構造函數)都聲明成partial、函數實現寫在.Impl.cs中
	public partial void Click1();
	
	///記得在這裏寫註釋
	public partial Task<nil> CallService(CT Ct);
}
```

`VmUserProfile.Impl.cs`:

```cs
namespace MyProj.Views.UserProfile;
/// .Impl.cs專門用來寫函數實現
public partial class VmUserProfile{
	public partial VmUserProfile(
		ISvcUser? SvcUser;
		ISvcUserCtx? SvcUserCtx;
	){
		this.SvcUser = SvcUser;
		this.SvcUserCtx = SvcUserCtx;
		this.Inited = true;
	}
	
	//無參非異步函數、用于給普通按鈕綁定、只涉及ViewModel內部狀態的修改 無耗時操作
	public partial void Click1(){
		Cnt1++;
	}
	
// 在ViewModel中調用後端服務示例
// 聲明爲異步函數、函數名不需特殊後綴、參數設爲CT Ct即可。
// 此函數用于給OpBtn綁定
	public partial async Task<nil> CallService(CT Ct){
		//由于注入的依賴都是可空類型、調用時需先判空。
		CheckInited();
		
		//防止UI卡頓
			await RunTask(async ()=>{
				var R = await SvcUser.ServeApi1(SvcUserCtx.GetUserCtx(), Ct);
		//示例: 如果要在子線程中修改UI，須這樣寫
				Dispatcher.UIThread.Post(()=>{
					this.Input += R+"";
				});
			},Ct);
		return NIL;
	}
	
}
```

注意:`InInited`, `CheckInit()`, 應當在Vm的基類中定義; `IMk<>`接口應當在項目中定義。若項目未定義則應請示用戶

## View規範

使用 `Tsinswreng.Avln.Dsl` 庫。

`ViewUserProfile.cs`:

```cs
namespace MyProj.Views.UserProfile;
using Tsinswreng.Avln.Dsl;
using Tsinswreng.Avln.Grid;
using Ctx = VmUserProfile;
///記得在這裏寫註釋
public partial class ViewUserProfile: AppViewBase<Ctx>{
	
	//View必須有且現有一個無參構造器
	public partial ViewSample();
	
	//此類型來自Tsinswreng.Avln.Grid;
	//大多數場景下我們用GridStack作爲視圖的根節點、
	//他本身不是Control的子類、內部包含了一個Grid屬性。
	//GridStack支持 或全爲行 或全爲列 的佈局 不建議同時設置行和例。
	//每次Add時會自動設置行號或列號 因此不要手動設計行/列號
	//優先使用 GridStack 或嵌套 GridStack、別用原生Grid 除非你要顯示手動設置行和列
	//IsRow:true表示全爲行的佈局。
	public GridStack Root = new(IsRow:true);
	
	//頁面中的關鍵控件都要獨立作爲類的public成員、以方便設計規劃與測試。
	//關鍵控件包括:
	//涉及輸入操作交互(如按鈕,輸入框),信息展示的(如文本框);
	//子模塊/子UserControl(其他的`ViewXxx`)。
	//聲明關鍵控件時、可使用具體類型(如 `public TextBox? _CtrlCnt1`;)
	//若暫時未確定類型也可以寫`Control?`或`object?`。
	//聲明爲可空、不需要寫get;set;
	///(寫註釋說明這個控件)
	public Control? _CtrlCnt1;
	
	///點擊按鈕後增加Cnt1的值
	public Button? _BtnAdd;
	
	//初始化視圖
	public partial void Render();
	
	//初始化樣式
	public partial void InitStyle();
	
	//(可選) 視圖加載完成後的回調函數、可在此函數中做一些初始化操作
	//耗時操作必須放在OnLoaded中、不能阻塞UI創建
	public partial void OnLoaded();
	
//當View中縮進層次過多旹、或子控件是一個相對獨立的邏輯單元、
//可將Render中的部分代碼抽到一個單獨的函數中、把控件返回出去、再在Render中調用
	public partial Control MkList();
	
	public partial Control MkBar();
}
```

`ViewUserProfile.Impl.cs`:

```cs
namespace MyProj.Views.UserProfile;
using Tsinswreng.Avln.Dsl;
using Tsinswreng.Avln.Grid;
using Ctx = VmUserProfile;
///記得在這裏寫註釋
public partial class ViewUserProfile: AppViewBase<Ctx>{
	
	
	public partial ViewSample(){
		Ctx = App.DiOrMk<Ctx>();
		InitStyle();
		Render();
		//(可選)
		Loaded += (s,e)=>{
			OnLoaded();
		}
	}
	
	public partial void OnLoaded(){
		
	}
	
	public partial void Render(){
		//所有ContentControl及子類、都用SetContent來初始化其Content、避免直接賦值
		this.SetContent(Root.Grid);
		Root.SetRowDefs([
			new(1, GUT.Auto),
			new(2, GUT.Star),
			new(30, GUT.Pixel),
			//....略
		]);
		
		
		//使用.A()方法添加子控件、並在匿名函數中初始化。
		//所有Panel及子類都有.A()方法。
		//能用.A()方法的就禁止手動.Children.Add
		//匿名函數中的形參要短、以一到兩個字母爲宜
		Root.A(new TextBox(), t=>{
			_CtrlCnt1 = t;//初始化關鍵控件這樣寫
			t.AcceptsReturn = true;
			//下面展示綁定寫法。須用AOT兼容的綁定寫法。不准用new Binding(字符串)
			//功能最全但不是最常用的寫法:
			t.Bind(
				TextBox.TextProperty
				,CBE.Mk<Ctx>(x=>x.Cnt1)//對應Ctx(即ViewModel)中的成員
				#if false //可加其他可選命名參數如:
				,Converter:
				,ConverterParameter:
				,Mode:
				#endif
			);
			
			//寫法二。比上面的寫法稍簡單點 但仍然不是最簡單的寫法 也不是最常用的寫法。
			//這種寫法一般用于綁定源不是Ctx的情況(如ItemTemplate中)。當綁定源是Ctx時優先用寫法三
			o.CBind<Ctx>(TextBox.TextProperty, x=>x.Cnt1);

			//寫法三。大部分情況下、綁定源都是Ctx 故此時則採用簡便寫法如下:
			Ctx.Bind(o, o=>o.Text, x=>x.Cnt1);
			//上面的第二個參數優先用o=>o.Text的寫法。如當o爲TextBox時o=>o.Text即等於TextBox.TextProperty。
			// 不得已時再用 類名.XxxProperty的寫法
			
			//所有綁定 不需要顯示指定BindingMode的 就別寫BindingMode !
		}).A(new Button(), b=>{
			_BtnAdd = b;//初始化關鍵控件
			//添加樣式類名
			b.Classes.Add(Cls.MenuBtn);
			//初始化ContentControl.Content時使用SetContent擴展方法、
			//並在匿名函數中初始化。不要直接給Content賦值
			b.SetContent(new TextBlock(), t=>{
				//項目要支持多語言本地化。所有UI文本禁止直接硬編碼。
				//這裏展示臨時硬編碼的寫法。具體I18n規範另尋他處。
				t.Text = Todo.I18n("點我加一")
			});
			
			//按鈕綁定同步非耗時事件的寫法
			b.Click += (s,e)=>{
				Ctx?.Click1();
			};
		
		})
		.A(new Border(), Bd=>{
			Bd.BorderBrush = Brushes.Red;
			Bd.BorderThickness = new(2); //能直接寫new(...)的、就不要寫 new TypeName(...)
			//Border必須使用SetChild、禁止使用 b.Child=xxx 的寫法
			Bd.SetChild(new ScrollViewer(), Sv=>{
				var Lg = new GridStack(IsRow:true);
				Sv.SetContent(Lg.Grid, _=>{
					Lg.SetRowDefs([
						new(1, GUT.Auto),
						new(1, GUT.Star),
					]);
					Lg.A(MkBar())
					.A(MkList())
					;
				});
			})
		})
		;
		//組織子控件並加入控件樹時、代碼塊的嵌套 要和 樹的邏輯結構 保持一致
		//如上方的示例的包裹層級是Border>ScrollViewer>GridStack>MkBar,MkList
		//則在代碼中嵌套關係也要保持一致、即Border縮進層級最少、往後依次增多
	}
	
	//列表寫法示例
	public partial Control MkList(){
		var R = new ItemsControl();
		R.SetItemTemplate<str>((ele, ns)=>{
			return new TextBox{Text=ele};
		}).SetItemsPanel(()=>{
			return new VirtualizingStackPanel();
		});
		R.Bind(R.PropItemsSource, CBE.Mk<Ctx>(x=>x.List));
		return R;
	}
	
	public partial Control MkBar(){
		//略
	}
	
	//樣式(非必選)示例
	//樣式簡單旹建議直接和控件一起初始化。有[需批量設置樣式]等需求旹再用Style
	
	public partial class Cls{//類名枚舉。禁止用字符串硬編碼類名
		public const string MenuBtn = nameof(MenuBtn);
	}
	
	public partial void InitStyle(){
		Styles.A(
			//傳統寫法。通常要寫完整的類名.XxxProperty、比較冗長且無編譯期類型檢查。不推薦
			new Style(x=>
				x.Is<Button>()
				.Class(Cls.MenuBtn)//禁止硬編碼字符串作類名
			).Set(
				VerticalAlignmentProperty
				,VAlign.Stretch
			).Set(//.Set也能且必須鏈式調用
				HorizontalAlignmentProperty
				,HAlign.Center
			)
		).A(
			Sty.Is<Button>(//Sty下有Is和OfType
				x=>x.Class(Cls.MenuBtn)
				//此寫法即等價于 new Style(x=>x.Is<Button>().Class(Cls.MenuBtn))、但更簡潔且有編譯期類型檢查
			).Set(
				//更推薦的簡便寫法、不用寫ClsName.XxxProperty
				x=>x.HorizontalAlignment, HAlign.Center
			).Set(
				x=>x.VerticalAlignment, VAlign.Stretch
			)
		).A(//Styles也必須使用.A鏈式調用
			Sty.Is<ContentControl>()
			.Set(
				//可以在Style中設置綁定
				x=>x.CornerRadius, CBE.Mk<Ctx>(
					x=>x.Cnt1,
					Converter: new FnConvtr<int, CornerRadius>(cnt=>{
						return new CornerRadius(cnt);
					})
				)
			)
		)
		;
	}

}
```

注: `Todo.I18n()`,`GUT`別名應在項目中定義、如未定義則應請示用戶

## 再次強調所該做與不該做

### 所該做

- 用`.A()`、`SetContent()`、`SetChild()` 等擴展方法來組織控件樹
- 加入控件樹時、在 lambda 中用 `o.Xxx = Yyy` 的方式初始化控件
- 代碼塊的嵌套層級要與實際控件樹結構保持一致
- 優先使用 `GridStack` 或嵌套 `GridStack` 組織佈局
- 使用 AOT 兼容的綁定寫法；綁定源是 `Ctx` 時優先用 `Ctx.Bind(...)`
- View 中的耗時初始化放到 `OnLoaded()`，不要阻塞 UI 創建
- 樣式有重複時抽到 `Styles` 中，並用 `Cls` 常量管理類名
- UI 文本走 I18n；字體大小、間距等優先走項目配置或統一常量
- 如果項目缺少本 skill 依賴的基礎設施，如 `IMk<>`、`Todo.I18n()`、綁定輔助器、View/Vm 基類，立即請示用戶
- 在ViewXxx中把關鍵控件提到public成員。

### 所不該做

- 不要手動寫 `panel.Children.Add(...)`；能用 `.A()` 的地方一律用 `.A()`
- 不要直接給 `ContentControl.Content`、`Border.Child` 等賦值；分別用 `SetContent()`、`SetChild()`
- 不要用 `new Binding("...")` 或其他字符串路徑綁定
- 不要在 ViewModel 層做視圖跳轉、操作 View 控件、耦合 View 層細節
- 不要把耗時操作寫進 View 構造器、同步按鈕事件、或直接阻塞 UI 線程
- 不要硬編碼 UI 文本、字體大小、樣式類名
- 不要爲了“拆函數而拆函數”；只有在 Render 嵌套過深或子區塊相對獨立時才抽 `MkXxx()`
- 不要手動設計 `GridStack` 子項的行號列號；讓其自動分配
- 不要在 `Xxx.cs` 中寫函數實現；聲明與實現必須分離到 `Xxx.Impl.cs`
- 不要在基礎設施是否存在、命名是否一致、規範是否適配當前項目這些問題上自行猜測；有疑問立即停下來問用戶
