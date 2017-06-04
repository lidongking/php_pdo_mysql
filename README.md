# php_pdo_mysql
## Firstly
Thank you very much for viewing my code. At first, there's an idea burning in my mind: "Origined mysql or class's functions is bothering for development". Then I want to wrapper the methods and functions to save the code lines and time, so I do.

## Secondly
Codes Coming ^_^...
``` php
class DB
{
    protected $dsn;
    protected $dbUser;
    protected $dbPass;
    protected $tablePrefix;
    protected $charset;
    protected $isPersistent;
    protected $isDebug;
    protected $pdo;
    protected $pdoStatement;

    protected $queryType;
    protected $sql;
    protected $where;
    protected $order;
    protected $limit;
    protected $data;
    protected $isExec;

    protected static $instances = array();

    private function __construct($dbKey)
    {
        global $CONF;
        $config = $CONF['DB'][$dbKey];
        $dbHost = $config['host'];
        $dbPort = isset($config['port']) && $config['port'] ? $config['port'] : '3306';
        $dbUser = $config['user'];
        $dbPass = $config['pass'];
        $dbName = $config['dbName'];
        $tablePrefix = isset($config['prefix']) ? $config['prefix'] : '';
        $charset = isset($config['charset']) ? strtolower(str_replace('-', '', $config['charset'])) : 'utf8';
        $isPersistent = empty($config['persistent']) ? false : true;
        $isDebug = empty($CONF['DB'][$dbKey]['debug']) ? false : true;

        if (empty($CONF['DB'][$dbKey]) || empty($dbHost) || empty($dbUser) || empty($dbPass) || empty($dbName))
        {
            throw new Exception('DATABASE $dbKey ' . $dbKey . ' is not good.');
        }
        $this->dsn = 'mysql:host=' . $dbHost . ';port=' . $dbPort . ';dbname=' . $dbName;
        $this->dbUser = $dbUser;
        $this->dbPass = $dbPass;
        $this->tablePrefix = $tablePrefix;
        $this->charset = $charset;
        $this->isPersistent = $isPersistent;
        $this->isDebug = $isDebug;
        $this->connect();
    }

    private function __clone()
    {
    }

    /**
     * 功    能：获取操作DB实例
     * 修改日期：2017-6-4
     *
     * @param string $dbKey 数据库标示Key
     *
     * @return mixed 当前DB实例
     */
    public static function getInstance($dbKey = 'default')
    {
        $dbKey = strtoupper($dbKey);
        if (empty(self::$instances[$dbKey]))
        {
            self::$instances[$dbKey] = new static($dbKey);
        }

        return self::$instances[$dbKey];
    }

    /**
     * 功    能：连接数据库
     * 修改日期：2017-6-4
     *
     * @return null 无返回
     */
    private function connect()
    {
        try
        {
            $options = array();
            if ($this->isPersistent)
            {
                $options[] = array(PDO::ATTR_PERSISTENT => true);
            }
            $this->pdo = new PDO($this->dsn, $this->dbUser, $this->dbPass, $options);
            $this->pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        }
        catch (PDOException $e)
        {
            $this->halt('ERROR:' . $e->getMessage());
        }
        if ($this->pdo)
        {
            // $this->pdo->setAttribute( PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION );
            $this->pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);
            $this->pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
            $this->pdo->exec("SET NAMES {$this->charset}");
        }
    }

    /**
     * 功    能：获取pdo对象,以便直接操作pdo
     * 修改日期：2017-6-4
     *
     * @return mixed pdo对象
     */
    public function getPdo()
    {
        return $this->pdo;
    }

    /**
     * 功    能：初始化参数
     * 修改日期：2017-6-4
     *
     * @return null 无返回
     */
    public function init()
    {
        $this->queryType = '';
        $this->sql = '';
        $this->where = '';
        $this->order = '';
        $this->limit = '';
        $this->data = array();
        $this->isExec = false;
    }

    /**
     * 功    能：
     * 修改日期：
     *
     * @param $table 表名
     * @param array $fileds 字段
     *
     * @return $this 返回当前对象，链式操作
     */
    public function select($table, $fileds = array())
    {
        $this->init();
        $this->queryType = 'S';
        $filedStr = empty($fileds) ? '`*`' : '`' . implode('`,`', $fileds) . '`';
        $this->sql = 'SELECT ' . $filedStr . ' FROM `' . $this->tablePrefix . $table . '`';

        return $this;
    }

    /**
     * 功    能：插入方法
     * 修改日期：2017-6-4
     *
     * @param $table 表名
     * @param $data 数据
     *
     * @return $this 返回当前对象，链式操作
     */
    public function insert($table, $data)
    {
        $this->init();
        $this->queryType = 'I';
        $first = current($data);
        if (is_array($first))
        {
            // 多行插入
            $fields = array_keys($first);
            $values = substr(str_repeat('?,', count($fields)), 0, -1);
            $valuesAll = substr(str_repeat('(' . $values . '),', count($data)), 0, -1);

            $this->sql = 'INSERT INTO `' . $this->tablePrefix . $table . '`(`' . implode('`,`', $fields) . '`) VALUES' . $valuesAll;
            foreach ($data as $key => $item)
            {
                foreach ($item as $kkey => $vval)
                {
                    $this->data[$kkey . '_' . $key] = $vval;
                }
            }
        }
        else
        {
            $fields = array_keys($data);
            $values = substr(str_repeat('?,', count($fields)), 0, -1);
            $this->sql = 'INSERT INTO `' . $this->tablePrefix . $table . '`(`' . implode('`,`', $fields) . '`) VALUES(' . $values . ')';
            //$this->data = array_values($data);
            foreach ($data as $key => $val)
            {
                $this->data[$key] = $val;
            }
        }

        return $this;
    }

    /**
     * 功    能：
     * 修改日期：
     *
     * @param String $table 表明
     * @param array $data 数据
     *
     * @return $this 返回当前对象，链式操作
     */
    public function update(String $table,Array $data)
    {
        $this->init();
        $this->queryType = 'U';
        $fields = array_keys($data);
        $this->sql = 'UPDATE `' . $this->tablePrefix . $table . '` SET ' . implode('=?', $fields) . '=?';
        $this->data = $data;

        return $this;
    }

    /**
     * 功    能：删除数据方法
     * 修改日期：2017-6-4
     *
     * @param String $table 表名
     *
     * @return $this 返回当前对象，链式操作
     */
    public function delete(String $table)
    {
        $this->init();
        $this->queryType = 'D';
        $this->sql = 'DELETE FROM `' . $this->tablePrefix . $table . '`';

        return $this;
    }

    /**
     * 功    能：获取where语句
     * 修改日期：2017-6-4
     *
     * @param $str where条件
     * @param null $parameter 参数绑定
     *
     * @return $this 返回当前对象，链式操作
     */
    public function where($str, $parameter = null)
    {
        if (null !== $parameter)
        {
            if (is_array($parameter))
            {
                $this->data += $parameter;
                // 根据实际传递的参数数目，替换in语句中的？，只能有一个in语句
                $c1 = substr_count($str, '?');
                $c2 = count($parameter);
                $replace = 'in(' . substr(str_repeat('?,', $c2 - $c1 + 1), 0, -1) . ')';
                $str = str_replace('in(?)', $replace, $str);
            }
            else
            {
                $this->data[] = $parameter;
            }
        }
        $this->where = ' WHERE ' . $str;

        return $this;
    }

    /**
     * 功    能：排序
     * 修改日期：2017-6-4
     *
     * @param $str order语句
     *
     * @return $this 返回当前对象，链式操作
     */
    public function order($str)
    {
        $this->order = ' ORDER BY ' . $str;

        return $this;
    }

    /**
     * 功    能：限定条数
     * 修改日期：2017-6-4
     *
     * @param int $offset 起始数据位置
     * @param int $size 返回条目
     *
     * @return $this 返回当前对象，链式操作
     */
    public function limit($offset = 0, $size = 10)
    {
        $this->limit = ' LIMIT ' . $offset . ',' . $size;

        return $this;
    }

    /**
     * 功    能：
     * 修改日期：2017-6-4
     *
     * @param $sql sql语句
     * @param null|array $parameter 参数
     *
     * @return $this 返回当前对象，链式操作
     */
    public function query($sql, $parameter = null)
    {
        $this->init();
        if (null !== $parameter)
        {
            if (is_array($parameter))
            {
                $this->data = $parameter;
                // 根据实际传递的参数数目，替换in语句中的？，只能有一个in语句
                $c1 = substr_count($sql, '?');
                $c2 = count($parameter);
                $replace = 'in(' . substr(str_repeat('?,', $c2 - $c1 + 1), 0, -1) . ')';
                $sql = str_replace('in(?)', $replace, $sql);
            }
            else
            {
                $this->data[] = $parameter;
            }
        }
        $sql = str_replace( '`_', '`' . $this->tablePrefix, $sql );
        $this->sql = $sql;

        return $this;
    }

    /**
     * 功    能：执行语句
     * 修改日期：2017-6-4
     *
     * @return bool 执行结果
     */
    public function execute()
    {
        if ($this->pdo)
        {
            switch ($this->queryType)
            {
                case 'S':
                    $this->sql .= $this->where . $this->order . $this->limit;
                    break;
                case 'U':
                    $this->sql .= $this->where;
                    break;
                case 'D':
                    $this->sql .= $this->where;
                    break;
            }
            if ($this->pdoStatement = $this->pdo->prepare($this->sql))
            {
                $pos = 1;
                foreach ($this->data as $key => $val)
                {
                    if (!$this->pdoStatement->bindValue($pos, $val))
                    {
                        return false;
                    }
                    $pos++;
                }
                if ($this->pdoStatement->execute())
                {
                    return true;
                }
            }
        }

        return false;
    }

    /**
     * 功    能：获取所有数据
     * 修改日期：2017-6-4
     *
     * @param int $style 返回数据类型,默认数据
     *
     * @return bool 结果
     */
    public function findAll($style = PDO::FETCH_ASSOC)
    {
        if ($this->execute())
        {
            return $this->pdoStatement->fetchAll($style);
        }

        return false;
    }

    /**
     * 功    能：获取一行数据
     * 修改日期：2017-6-4
     *
     * @param int $style 返回数据类型,默认数据
     *
     * @return array|bool 结果
     */
    public function find($style = PDO::FETCH_ASSOC)
    {
        $res = false;
        if ($this->execute())
        {
            $res = $this->pdoStatement->fetch($style);
            $res = $res === false ? array() : $res;
        }

        return $res;
    }

    /**
     * 功    能：返回第1行第1列的值
     * 修改日期：2017-6-4
     *
     * @return bool|null 返回值
     */
    public function findCell()
    {
        $res = false;
        if ($this->execute())
        {
            $res = $this->pdoStatement->fetchColumn();
            $res = $res === false ? null : $res;
        }

        return $res;
    }

    /**
     * 功    能：执行事务操作
     * 修改日期：2017-6-4
     *
     * @return mixed 结果
     */
    public function beginTransaction()
    {
        return $this->pdo->beginTransaction();
    }

    /**
     * 功    能：事务提交
     * 修改日期：2017-6-4
     *
     * @return mixed 结果
     */
    public function commit()
    {
        return $this->pdo->commit();
    }

    /**
     * 功    能：事务回滚
     * 修改日期：2017-6-4
     *
     * @return mixed 结果
     */
    public function rollback()
    {
        return $this->pdo->rollback();
    }

    /**
     * 功    能：是否在事务中
     * 修改日期：2017-6-4
     *
     * @return mixed 结果
     */
    public function inTransaction()
    {
        return $this->pdo->inTransaction();
    }

    /**
     * 功    能：获取上次插入id
     * 修改日期：2017-6-4
     *
     * @return bool|int 上次插入id
     */
    public function lastInsertId()
    {
        if ($this->execute())
        {
            return $this->pdo->lastInsertId();
        }

        return false;
    }

    /**
     * 功    能：返回影响行数
     * 修改日期：2017-6-4
     *
     * @return bool|int 行数
     */
    public function affectedRows()
    {
        if ($this->execute())
        {
            return $this->pdoStatement->rowCount();
        }

        return false;
    }

    /**
     * 功    能：输出信息
     * 修改日期：2017-6-4
     *
     * @param $msg 信息
     */
    public function halt($msg)
    {
        if ($this->isDebug)
        {
            echo $msg;
        }
    }
}
```
