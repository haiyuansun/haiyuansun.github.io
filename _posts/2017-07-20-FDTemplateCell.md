---
layout:     post
title:      FDTemplateLayoutCell 源码分析
category: blog
description: FDTemplateLayoutCell的源码分析，注重对于缓存的实现。
---

# FDTemplateLayoutCell 源码分析

##接口分析
对于FDTemplateLayoutCell这套代码而言，接口设计比较简单易用，所以我们首先对于其缓存接口分析，关于其他不重要的接口暂时略过。在`heightForRowAtIndexPath`中我们可以看到FDTemplateLayoutCell提供给用户返回行高的方法：
```
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    /*取出当前用户选择的缓存方式*/
    FDSimulatedCacheMode mode = self.cacheModeSegmentControl.selectedSegmentIndex;
    switch (mode) {
        /*无缓存*/
        case FDSimulatedCacheModeNone:
            return [tableView fd_heightForCellWithIdentifier:@"FDFeedCell" configuration:^(FDFeedCell *cell) {
                [self configureCell:cell atIndexPath:indexPath];
            }];
        /*根据indexPath缓存*/
        case FDSimulatedCacheModeCacheByIndexPath:
            return [tableView fd_heightForCellWithIdentifier:@"FDFeedCell" cacheByIndexPath:indexPath configuration:^(FDFeedCell *cell) {
                [self configureCell:cell atIndexPath:indexPath];
            }];
        /*根据模型的key缓存*/
        case FDSimulatedCacheModeCacheByKey: {
            FDFeedEntity *entity = self.feedEntitySections[indexPath.section][indexPath.row];

            return [tableView fd_heightForCellWithIdentifier:@"FDFeedCell" cacheByKey:entity.identifier configuration:^(FDFeedCell *cell) {
                [self configureCell:cell atIndexPath:indexPath];
            }];
        };
        default:
            break;
    }
}
```
注意到，每一个方法都传入了一个名为`configuration`的block进行回调`- (void)configureCell:(FDFeedCell *)cell atIndexPath:(NSIndexPath *)indexPath`方法。 这个方法的具体实现如下：
```
- (void)configureCell:(FDFeedCell *)cell atIndexPath:(NSIndexPath *)indexPath {
    cell.fd_enforceFrameLayout = NO; // Enable to use "-sizeThatFits:"
    if (indexPath.row % 2 == 0) {
        cell.accessoryType = UITableViewCellAccessoryDisclosureIndicator;
    } else {
        cell.accessoryType = UITableViewCellAccessoryCheckmark;
    }
    cell.entity = self.feedEntitySections[indexPath.section][indexPath.row];
}
```
实际上这是对Cell中的模型复制抽取的一个方法。
## 内部分析
`fd_heightForCellWithIdentifier` 内部的实现逻辑可以使用以下代码进行表示：
```
- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier cacheBySomeMehthod:(id *)methodId configuration:(void (^)(id cell))configuration {
    //传入一种缓存方式
	//这里的methodId 可以是key 或者 indexPath
	//接下来以indexPath为例
	if method != none{
		if (!identifier || !indexPath) {
			return 0;
		}
    
		// Hit cache
		if ([self.fd_indexPathHeightCache existsHeightAtIndexPath:indexPath]) {/*查找是否命中缓存*/
               /*缓存命中，取出缓存并返回*/
			[self fd_debugLog:[NSString stringWithFormat:@"hit cache by index path[%@:%@] - %@", @(indexPath.section), @(indexPath.row), @([self.fd_indexPathHeightCache heightForIndexPath:indexPath])]];
			return [self.fd_indexPathHeightCache heightForIndexPath:indexPath];
		}
		/*缓存未命中，则计算根据当前Cell计算行高，并对计算结果根据indexPath缓存*/
		CGFloat height = [self fd_heightForCellWithIdentifier:identifier configuration:configuration];
		[self.fd_indexPathHeightCache cacheHeight:height byIndexPath:indexPath];
		[self fd_debugLog:[NSString stringWithFormat: @"cached by index path[%@:%@] - %@", @(indexPath.section), @(indexPath.row), @(height)]];
		
	}else{/*如果不需要缓存行高 则直接生成templateLayoutCell 利用templateLayoutCell计算行高后返回*/
                height = fd_heightForCellWithIdentifier 方法计算结果
	}

    return height;
}

- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier configuration:(void (^)(id cell))configuration {
    if (!identifier) {
        return 0;
    }
    
    UITableViewCell *templateLayoutCell = [self fd_templateCellForReuseIdentifier:identifier];
    
    // Manually calls to ensure consistent behavior with actual cells. (that are displayed on screen)
    [templateLayoutCell prepareForReuse];
    
    // Customize and provide content for our template cell.
    if (configuration) {
        configuration(templateLayoutCell);
    }
    
    return [self fd_systemFittingHeightForConfiguratedCell:templateLayoutCell];
}

```
可以看到，当传入参数为需要缓存时，会先调用`FDIndexPathHeightCache`类的
 `- (BOOL)existsHeightAtIndexPath:(NSIndexPath *)indexPath` 
方法判断传入的indexPath是否已经缓存过行高，如果有则直接取出；如果没有，则进行下一步：调用
`- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier configuration:(void (^)(id cell))configuration` 
方法进行行高计算。关于`FDIndexPathHeightCache`类和`FDKeyedHeightCache`类的缓存实现后面会具体介绍，现在继续深入`templateLayoutCell `模块的设计。
### 生成templateCell并根据这个Cell计算行高

`- (CGFloat)fd_heightForCellWithIdentifier:(NSString *)identifier configuration:(void (^)(id cell))configuration` 
在上面的方法中，identifier 和 block 都作为入参，block自然是作为回调使用，identifier的作用在接下来的过程中很重要。
**该方法调用`fd_templateCellForReuseIdentifier`返回一个UITableViewCell类型的cell。这个cell在后续被函数`fd_systemFittingHeightForConfiguratedCell`调用作为入参进行行高的计算。 **

好了，到现在我们大致知道这个框架到现在为止做了一些什么。

1. 判断是否需要缓存UITableViewCell的行高。
2. 如果不需要，直接前往第四步。
3. 根据indexPath或者key查找是否有计算并缓存过的行高，如果有，前往第五步。
4. 生成templateCell，根据这个Cell和传入的identifier进行行高的计算。
5. 返回行高给控制器的`heightForRow`方法，如果需要缓存则根据对应的indexPath或者key进行缓存。

###行高计算
`fd_systemFittingHeightForConfiguratedCell`方法中templateCell作为入参参与行高计算，而不是用于显示在屏幕上。

```
#pragma mark 计算Cell的行高
- (CGFloat)fd_systemFittingHeightForConfiguratedCell:(UITableViewCell *)cell {
    CGFloat contentViewWidth = CGRectGetWidth(self.frame);
    
    CGRect cellBounds = cell.bounds;
    cellBounds.size.width = contentViewWidth;
    cell.bounds = cellBounds;
    
    CGFloat accessroyWidth = 0;
    // If a cell has accessory view or system accessory type, its content view's width is smaller
    // than cell's by some fixed values.
    if (cell.accessoryView) {
        accessroyWidth = 16 + CGRectGetWidth(cell.accessoryView.frame);
    } else {
        /*生成一个字典存储不同accessoryType对应的宽度*/
        static const CGFloat systemAccessoryWidths[] = {
            [UITableViewCellAccessoryNone] = 0,
            [UITableViewCellAccessoryDisclosureIndicator] = 34,
            [UITableViewCellAccessoryDetailDisclosureButton] = 68,
            [UITableViewCellAccessoryCheckmark] = 40,
            [UITableViewCellAccessoryDetailButton] = 48
        };
        accessroyWidth = systemAccessoryWidths[cell.accessoryType];
    }
    contentViewWidth -= accessroyWidth;

    
    // If not using auto layout, you have to override "-sizeThatFits:" to provide a fitting size by yourself.
    // This is the same height calculation passes used in iOS8 self-sizing cell's implementation.
    //
    // 1. Try "- systemLayoutSizeFittingSize:" first. (skip this step if 'fd_enforceFrameLayout' set to YES.)
    // 2. Warning once if step 1 still returns 0 when using AutoLayout
    // 3. Try "- sizeThatFits:" if step 1 returns 0
    // 4. Use a valid height or default row height (44) if not exist one
    
    CGFloat fittingHeight = 0;
if (!cell.fd_enforceFrameLayout && contentViewWidth > 0) {
        // Add a hard width constraint to make dynamic content views (like labels) expand vertically instead
        // of growing horizontally, in a flow-layout manner.
//以下添加约束的代码省略
// Auto layout engine does its math
        fittingHeight = [cell.contentView systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;
        
}
    if (fittingHeight == 0) {
        // Try '- sizeThatFits:' for frame layout.
        // Note: fitting height should not include separator view.
        fittingHeight = [cell sizeThatFits:CGSizeMake(contentViewWidth, 0)].height;
        
        [self fd_debugLog:[NSString stringWithFormat:@"calculate using sizeThatFits - %@", @(fittingHeight)]];
    }
  
    // Still zero height after all above.
    if (fittingHeight == 0) {
        // Use default row height.
        fittingHeight = 44;
    }
    
    // Add 1px extra space for separator line if needed, simulating default UITableViewCell.
    if (self.separatorStyle != UITableViewCellSeparatorStyleNone) {
        fittingHeight += 1.0 / [UIScreen mainScreen].scale;
    }
    
    return fittingHeight;

```
这个方法内部主要逻辑如下：
1. 首先判断用户是否使用frameLayout来进行界面的布局，如果不是，则默认用户使用autolayout进行布局，执行第二步；如果是，则执行第三步。
2. 添加额外约束，然后使用`- (CGSize)systemLayoutSizeFittingSize:(CGSize)targetSize `方法自动计算行高，如果`fittingHeight`返回0，则继续执行第三步。
3. 使用**重载**方法`- (CGSize)sizeThatFits:(CGSize)size `手动计算行高。这个方法为作者自己实现重载的方法。
4. 如果第三步仍然得不到结果，返回默认行高44。

###自动的缓存失效机制
>无须担心你数据源的变化引起的缓存失效，当调用如-reloadData，-deleteRowsAtIndexPaths:withRowAnimation:等任何一个触发 UITableView 刷新机制的方法时，已有的高度缓存将以最小的代价执行失效。如删除一个 indexPath 为 [0:5] 的 cell 时，[0:0] ~ [0:4] 的高度缓存不受影响，而 [0:5] 后面所有的缓存值都向前移动一个位置。自动缓存失效机制对 UITableView 的 9 个公有 API 都进行了分别的处理，以保证没有一次多余的高度计算。

FDTemplateCell中使用rumtime中的Method Swizzling将`UITableView`内部的`reloadData`、`inserSections:withRowAnimation:`、`deleteRowsAtIndexPaths:withRowAnimation:`等方法替换成自己的方法。从而达到以上叙述的目的。其实现方法比较容易明白，其实就是将缓存数组/字典中对应`indexPath`的缓存进行操作，而不影响其他内容。例如在`deleteRowsAtIndexPaths:withRowAnimation:`中：
```
    if (self.fd_indexPathHeightCache.automaticallyInvalidateEnabled) {
        [self.fd_indexPathHeightCache buildCachesAtIndexPathsIfNeeded:indexPaths];
        /*mutableIndexSetsToRemove中的key是section，对应的值为row*/
        NSMutableDictionary<NSNumber *, NSMutableIndexSet *> *mutableIndexSetsToRemove = [NSMutableDictionary dictionary];
        
        [indexPaths enumerateObjectsUsingBlock:^(NSIndexPath *indexPath, NSUInteger idx, BOOL *stop) {
            NSMutableIndexSet *mutableIndexSet = mutableIndexSetsToRemove[@(indexPath.section)];
            if (!mutableIndexSet) {
                mutableIndexSet = [NSMutableIndexSet indexSet]; // return indexSet with no members
                mutableIndexSetsToRemove[@(indexPath.section)] = mutableIndexSet;
            }
            [mutableIndexSet addIndex:indexPath.row];
        }];
        
        [mutableIndexSetsToRemove enumerateKeysAndObjectsUsingBlock:^(NSNumber *key, NSIndexSet *indexSet, BOOL *stop) {
            [self.fd_indexPathHeightCache enumerateAllOrientationsUsingBlock:^(FDIndexPathHeightsBySection *heightsBySection) {
                
                /*heightsBySection就是缓存行高用的数组*/
                /*typedef NSMutableArray<NSMutableArray<NSNumber *> *> FDIndexPathHeightsBySection;*/
                /*在每个section中删除对应需要删除的row*/
                [heightsBySection[key.integerValue] removeObjectsAtIndexes:indexSet];
            }];
        }];
    }
```
###关于使用RunLoop进行行高预缓存功能
在[作者的博客](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)中有提到一套利用runloop进行行高预缓存的方法，但是似乎已经被废弃，而且在源码中也没有找到。