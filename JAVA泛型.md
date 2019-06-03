# JAVA泛型 #
[泛型详解](泛型详解 "https://blog.csdn.net/s10461/article/details/53941091")

##1.为什么会出现泛型？ ##

举个例子，ArrayList.add()可以添加任何类型，取出来的时候，需要我们自己强转。也即是说存进去的时候我们知道它的具体类型，取出来的时候需要自己判断，有时候还经常出现错误，为了解决这类问题，jdk1.5提供泛型。

泛型即参数化类型，顾名思义，将原来的具体类型 参数化，将类型定义为参数形式。在使用的时候传入具体类型。

##2.泛型的本质  ##
是为了参数化类型（在不创建新的类型的情况下，通过泛型指定的不同类型来控制形参具体限制的类型）。也就是说在泛型使用过程中，操作的数据类型被指定为一个参数，这种参数类型可以用在类、接口和方法中，分别被称为泛型类、泛型接口、泛型方法。

##3.泛型的特性  ##
泛型只在编译阶段有效。

Java中的泛型，只在编译阶段有效。在编译过程中，正确检验泛型结果后，会将泛型的相关信息擦出，并且在对象进入和离开方法的边界处添加类型检查和类型转换的方法。也就是说，泛型信息不会进入到运行时阶段。
##4.泛型的使用 ##

三种使用方式、泛型类、泛型接口、泛型方法

**泛型类**

使用格式：

    class 类名称 <泛型标识：可以随便写任意标识号>{
	  private 泛型标识 /*（成员变量类型）*/ var; 
	  .....
	
	  }
	}

如果使用泛型类时不申明具体类型，那么该泛型类内定义的泛型方法，或泛型类型成员变量可以是任意类型。看下下面两段代码：

第一个类：

    public abstract class BaseMultiAdapter<T extends MultiItemEntity> 
	extends RecyclerView.Adapter<SuperViewHolder> {
	protected List<T> mDataList = new ArrayList<>();

第二个类：

    public class ExpandableItemAdapter extends
	 BaseMultiAdapter<MultiItemEntity> {

ExpandableItemAdapter类是BaseMultiAdapter的子类，在使用它的时候给它制定了类型MultiItemEntity类型，那么mDataList里只能存储MultiItemEntity类型的参数。

你看我给他传入别的类型就会报如下错误；

    (java.util.Collection<com.dkp.viewdemo.expandablelist.MultiItemEntity>)
	in BaseMultiAdapter cannot be applied to
	(java.util.ArrayList<java.lang.Integer>)
从而验证了我们以上的观点。

泛型接口和泛型类相似，不再赘述。

##5.通配符 ？ ##

类型通配符一般是使用？代替具体的类型实参，直白点的意思就是，此处的？和Number、String、Integer一样都是一种实际的类型，可以把？看成所有类型的父类。是一种真实的类型。

##6.泛型方法  ##

泛型类，是在实例化类的时候指明泛型的具体类型；泛型方法，是在调用方法的时候指明泛型的具体类型

	/**
	 * 泛型方法的基本介绍
	 * @param tClass 传入的泛型实参
	 * @return T 返回值为T类型
	 * 说明：
	 *     1）public 与 返回值中间<T>非常重要，可以理解为声明此方法为泛型方法。
	 *     2）只有声明了<T>的方法才是泛型方法，泛型类中的使用了泛型的成员方法
	 *     并不是泛型方法。
	 *     3）<T>表明该方法将使用泛型类型T，此时才可以在方法中使用泛型类型T。
	 *     4）与泛型类的定义一样，此处T可以随便写为任意标识，
	 *     常见的如T、E、K、V等形式的参数常用于表示泛型。
	 */
	
	public <T> T genericMethod(Class<T> tClass)throws InstantiationException ,
	  IllegalAccessException{
	        T instance = tClass.newInstance();
	        return instance;
	}
### 6.1泛型方法的基本用法 ###

    public class GenericTest {
    //这个类是个泛型类，在上面已经介绍过
    public class Generic<T>{     
        private T key;

        public Generic(T key) {
            this.key = key;
        }

        //我想说的其实是这个，虽然在方法中使用了泛型，但是这并不是一个泛型方法。
        //这只是类中一个普通的成员方法，只不过他的返回值是在声明泛型类已经声明过的泛型。
        //所以在这个方法中才可以继续使用 T 这个泛型。
        public T getKey(){
            return key;
        }

        /**
         * 这个方法显然是有问题的，在编译器会给我们提示这样的错误信息"cannot reslove symbol E"
         * 因为在类的声明中并未声明泛型E，所以在使用E做形参和返回值类型时，编译器会无法识别。
        public E setKey(E key){
             this.key = keu
        }
        */
    }

    /** 
     * 这才是一个真正的泛型方法。
     * 首先在public与返回值之间的<T>必不可少，这表明这是一个泛型方法，并且声明了一个泛型T
     * 这个T可以出现在这个泛型方法的任意位置.
     * 泛型的数量也可以为任意多个 
     *    如：public <T,K> K showKeyName(Generic<T> container){
     *        ...
     *        }
     */
    public <T> T showKeyName(Generic<T> container){
        System.out.println("container key :" + container.getKey());
        //当然这个例子举的不太合适，只是为了说明泛型方法的特性。
        T test = container.getKey();
        return test;
    }

    //这也不是一个泛型方法，这就是一个普通的方法，只是使用了Generic<Number>这个泛型类做形参而已。
    public void showKeyValue1(Generic<Number> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }

    //这也不是一个泛型方法，这也是一个普通的方法，只不过使用了泛型通配符?
    //同时这也印证了泛型通配符章节所描述的，?是一种类型实参，可以看做为Number等所有类的父类
    public void showKeyValue2(Generic<?> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }

     /**
     * 这个方法是有问题的，编译器会为我们提示错误信息："UnKnown class 'E' "
     * 虽然我们声明了<T>,也表明了这是一个可以处理泛型的类型的泛型方法。
     * 但是只声明了泛型类型T，并未声明泛型类型E，因此编译器并不知道该如何处理E这个类型。
    public <T> T showKeyName(Generic<E> container){
        ...
    }  
    */

    /**
     * 这个方法也是有问题的，编译器会为我们提示错误信息："UnKnown class 'T' "
     * 对于编译器来说T这个类型并未项目中声明过，因此编译也不知道该如何编译这个类。
     * 所以这也不是一个正确的泛型方法声明。
    public void showkey(T genericObj){

    }
    */

我们可以在泛型类中声明一个泛型方法如下：

    //在泛型类中声明了一个泛型方法，
	//使用泛型E，这种泛型E可以为任意类型。可以类型与T相同，也可以不同。
	//由于泛型方法在声明的时候会声明泛型<E>，因此即使在泛型类中并未声明泛型E，
	//编译器也能够正确识别泛型方法中识别的泛型。
	public <E> void show_3(E t){
		System.out.println(t.toString());
	}

### 6.2泛型方法与可变参数 ###

    public <T> void printMsg( T... args){
	    for(T t : args){
	        Log.d("泛型测试","t is " + t);
	    }
	}

### 6.3静态方法与泛型 ###
**静态方法无法访问类上定义的泛型；如果静态方法操作的引用数据类型不确定的时候，必须要将泛型定义在方法上**

**即：如果静态方法要使用泛型的话，必须将静态方法也定义成泛型方法**

    public class StaticGenerator<T> {
    ....
    ....
    /**
     * 如果在类中定义使用泛型的静态方法，需要添加额外的泛型声明（将这个方法定义成泛型方法）
     * 即使静态方法要使用泛型类中已经声明过的泛型也不可以。
     * 如：public static void show(T t){..},此时编译器会提示错误信息：
          "StaticGenerator cannot be refrenced from static context"
     */
    public static <T> void show(T t){

    }
	}

泛型方法使用原则：


> 无论何时，如果你能做到，你就该尽量使用泛型方法。也就是说，如果使用泛型方法将整个类泛型化，那么就应该使用泛型方法。另外对于一个static的方法而已，无法访问泛型类型的参数。所以如果static方法要使用泛型能力，就必须使其成为泛型方法。

### 6.3泛型的上下边界 ###

    public class Generic<T extends Number>{
    private T key;

    public Generic(T key) {
        this.key = key;
    }

    public T getKey(){
        return key;
    }
	}

？extends T 限制元素类型的上限，上限是T 只能传T的子类

？super   T 限制元素类型的下限，下限是T 只能传父类

泛型方法：

    //在泛型方法中添加上下边界限制的时候，必须在权限声明与返回值之间的<T>上添加上
	//下边界,即在泛型声明的时候添加
	//public <T> T showKeyName(Generic<T extends Number> container)，
	//编译器会报错："Unexpected bound"

	public <T extends Number> T showKeyName(Generic<T> container){
	    System.out.println("container key :" + container.getKey());
	    T test = container.getKey();
	    return test;
	}
> 通过上面的两个例子可以看出：泛型的上下边界添加，必须与泛型的声明在一起 。

### 6.4泛型数组 ###
**下面采用通配符的方式是被允许的:数组的类型不可以是类型变量，除非是采用通配符的方式，因为对于通配符的方式，最后取出数据是要做显式的类型转换的。**

    List<?>[] lsa = new List<?>[10]; // OK, array of unbounded wildcard type.    
	Object o = lsa;    
	Object[] oa = (Object[]) o;    
	List<Integer> li = new ArrayList<Integer>();    
	li.add(new Integer(3));    
	oa[1] = li; // Correct.    
	Integer i = (Integer) lsa[1].get(0); // OK 


