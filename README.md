# 工作日计算方式 --->由mysql完成

1.需要创建一个假期及加班表
CREATE TABLE `holiday` (
  `ID` varchar(200) NOT NULL DEFAULT '',
  `holidaydate` date DEFAULT NULL COMMENT '日期',
  `type` int(11) DEFAULT NULL COMMENT '1为节假日  2为调休日  ',
  `create_by` varchar(64) DEFAULT NULL COMMENT '创建者',
  `create_date` datetime DEFAULT NULL COMMENT '创建时间',
  `update_date` datetime DEFAULT NULL COMMENT '更新时间',
  `remarks` varchar(225) DEFAULT NULL COMMENT '备注',
  `del_flag` char(1) DEFAULT NULL COMMENT '删除标记',
  PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

2.项目使用节点表

CREATE TABLE `note_type` (
  `ID` varchar(200) NOT NULL DEFAULT '' COMMENT '主键',
  `NT_NAME` varchar(200) DEFAULT NULL COMMENT '环节名称',
  `NT_PARENTID` varchar(200) DEFAULT NULL COMMENT '父节点ID',
  `NT_DETP` int(11) DEFAULT NULL COMMENT '深度',
  `NT_ORDER` int(11) DEFAULT NULL COMMENT '排序',
  `NT1` varchar(200) DEFAULT NULL COMMENT '父节点一阶段名称',
  `NT2` varchar(200) DEFAULT NULL COMMENT '父节点二阶段名称',
  `NT3` varchar(200) DEFAULT NULL COMMENT '父节点三阶段名称',
  PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8

3.主表
CREATE TABLE `flowing` (
  `id` varchar(200) NOT NULL DEFAULT '' COMMENT '主键',
  `fl_xmdw` varchar(200) DEFAULT NULL COMMENT '项目单位',
  `fl_xmfl` varchar(200) DEFAULT NULL COMMENT '项目管理分类',
  `fl_xmmc` varchar(200) DEFAULT NULL COMMENT '项目名称',
  `fl_xmid` varchar(200) NOT NULL COMMENT '项目id',
  `fl_name` varchar(200) DEFAULT NULL COMMENT '环节名称',
  `fl_nid` varchar(200) NOT NULL COMMENT '环节id',
  `fl_blrid` varchar(200) DEFAULT NULL COMMENT '办理人id',
  `fl_path` varchar(200) DEFAULT NULL COMMENT '办理路径',
  `fl_day` varchar(200) DEFAULT NULL COMMENT '超期时限(工作日)',
  `fl_isend` int(11) DEFAULT NULL COMMENT '状态',
  `fl_zt` int(11) DEFAULT NULL COMMENT '是否督办(为null则未督办,上次督办时间)',
  `fl1` varchar(200) DEFAULT NULL COMMENT '备用1',
  `fl2` varchar(200) DEFAULT NULL COMMENT '备用2',
  `create_by` varchar(64) DEFAULT NULL COMMENT '创建者',
  `create_date` datetime DEFAULT NULL COMMENT '创建时间',
  `update_by` varchar(64) DEFAULT NULL COMMENT '更新者',
  `update_date` datetime DEFAULT NULL COMMENT '更新时间',
  `remarks` varchar(225) DEFAULT NULL COMMENT '备注',
  `del_flag` char(1) DEFAULT NULL COMMENT '删除标记',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='待办表'

思路：
  主表有一个该条信息的生成时间(update_date),
      一个工作日的时间(fl_day),
      剩下就是计算真正的工作日到期时间
      到期时间 = update_date + fl_day + 在update_date至update_date+fl_day之间周末数量 + 在update_date至update_date+fl_day之间节假日数量 - 在update_date至update_date+fl_day之间调班上班的日子
sql语句：
SELECT * FROM (SELECT a.id AS "id",
				a.fl_xmdw AS "flXmdw",
				a.fl_xmfl AS "flXmfl",
				a.fl_xmmc AS "flXmmc",
				a.fl_xmid AS "flXmid",
				a.fl_name AS "flName",
				a.fl_nid AS "flNid",
				a.fl_blrid AS "flBlrid",
				a.fl_path AS "flPath",
				a.fl_day AS "flDay",
				a.fl_isend AS "flIsend",
				a.fl_zt AS "flZt",
				a.fl1 AS "fl1",
				a.fl2 AS "fl2",
				a.create_by AS "createBy.id",
				a.create_date AS "createDate",
				a.update_by AS "updateBy.id",
				a.update_date AS "updateDate",
				a.remarks AS "remarks",
				a.del_flag AS "delFlag",
				CONVERT(a.fl_day/7,DECIMAL(10,0)) AS weeks,
				IF(a.fl_day%7=0
					,a.fl_day-2*(a.fl_day/7)
					,IF(DATE_FORMAT(a.update_date, "%w")>DATE_FORMAT(FROM_UNIXTIME(UNIX_TIMESTAMP(a.update_date)+(a.fl_day*24*60*60)),      "%w")
						,a.fl_day-1*(CONVERT(a.fl_day/7,DECIMAL(10,0))+1)
						,IF(DATE_FORMAT(a.update_date, "%w")<DATE_FORMAT(FROM_UNIXTIME(UNIX_TIMESTAMP(a.update_date)+(a.fl_day*24*60*60)), "%w")
							,IF(DATE_FORMAT(FROM_UNIXTIME(UNIX_TIMESTAMP(a.update_date)+(a.fl_day*24*60*60)), "%w") = 6
								,a.fl_day - 2 * CONVERT(a.fl_day/7,DECIMAL(10,0)) - 1
								,a.fl_day - 2 * CONVERT(a.fl_day/7,DECIMAL(10,0))
							   )
							,IF(DATE_FORMAT(a.update_date, "%w") = 6 || DATE_FORMAT(a.update_date, "%w") = 0
								,a.fl_day - 2 * CONVERT(a.fl_day/7,DECIMAL(10,0)) - 1
								,a.fl_day - 2 * CONVERT(a.fl_day/7,DECIMAL(10,0))
							   )
						    )
					   )
				    ) AS '工作日',
			    (SELECT COUNT(*) FROM holiday h WHERE  TYPE =1
				AND holidaydate > a.update_date 
				AND holidaydate <= FROM_UNIXTIME(UNIX_TIMESTAMP(a.update_date)+(a.fl_day*24*60*60))
				)*24*60*60 AS '节假日',
				(SELECT COUNT(*) FROM holiday h WHERE  TYPE =2
					AND holidaydate > a.update_date
					AND holidaydate <= FROM_UNIXTIME(UNIX_TIMESTAMP(a.update_date)+(a.fl_day*24*60*60))
				)*24*60*60 AS '调休日',
			FROM_UNIXTIME(UNIX_TIMESTAMP(a.update_date)
     <!--计算周末天数(包括周六周天)-->
				+IF(a.fl_day%7=0
					,a.fl_day-2*(a.fl_day/7)
					,IF(DATE_FORMAT(a.update_date, "%w")>DATE_FORMAT(FROM_UNIXTIME(UNIX_TIMESTAMP(a.update_date)+(a.fl_day*24*60*60)), "%w")
						,a.fl_day-1*(CONVERT(a.fl_day/7,DECIMAL(10,0))+1)
						,IF(DATE_FORMAT(a.update_date, "%w")<DATE_FORMAT(FROM_UNIXTIME(UNIX_TIMESTAMP(a.update_date)+(a.fl_day*24*60*60)), "%w")
							,IF(DATE_FORMAT(FROM_UNIXTIME(UNIX_TIMESTAMP(a.update_date)+(a.fl_day*24*60*60)), "%w") = 6
								,a.fl_day - 2 * CONVERT(a.fl_day/7,DECIMAL(10,0)) - 1
								,a.fl_day - 2 * CONVERT(a.fl_day/7,DECIMAL(10,0))
							   )
							,IF(DATE_FORMAT(a.update_date, "%w") = 6 || DATE_FORMAT(a.update_date, "%w") = 0
								,a.fl_day - 2 * CONVERT(a.fl_day/7,DECIMAL(10,0)) - 1
								,a.fl_day - 2 * CONVERT(a.fl_day/7,DECIMAL(10,0))
							   )
						    )
					   )
				    )*24*60*60
        <!--计算假日数量-->
				+(SELECT COUNT(*) FROM holiday h WHERE  TYPE =1
					AND holidaydate > a.update_date 
					AND holidaydate <= FROM_UNIXTIME(UNIX_TIMESTAMP(a.update_date)+(a.fl_day*24*60*60))
					AND DATE_FORMAT(holidaydate, "%w") != 6
					AND DATE_FORMAT(holidaydate, "%w") != 0
				)*24*60*60
        <!--调班上班-->
				-(SELECT COUNT(*) FROM holiday h WHERE  TYPE =2
					AND holidaydate > a.update_date
					AND holidaydate <= FROM_UNIXTIME(UNIX_TIMESTAMP(a.update_date)+(a.fl_day*24*60*60))
				)*24*60*60
			   ) AS alway_date
			 FROM flowing a,note_type nt WHERE a.fl_nid=nt.id
			 ) AS feh
 WHERE 1 = 1
 AND feh.alway_date < '2018-10-16 00:00:00' 

