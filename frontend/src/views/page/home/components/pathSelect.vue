<template>
	<div class="pathSelect">
		<el-dialog top="8vh" :close-on-click-modal="false" :visible.sync="dialogShow" title="选择目录" width="520px"
			:before-close="closeShow" :append-to-body="true">
			<el-tree :props="props" :load="loadNode" lazy :highlight-current="true" :check-on-click-node="true"
				@node-click="nodeClick" v-if="dialogShow">
			</el-tree>
			<span slot="footer" class="dialog-footer">
				<el-button @click="closeShow">取 消</el-button>
				<el-button type="primary" @click="submit" :loading="submitLoading"
					:disabled="cuPath == null">{{cuPath == null ? '请先选择路径' : '确 定'}}</el-button>
			</span>
		</el-dialog>
	</div>
</template>

<script>
	import {
		alistGetPath
	} from "@/api/job";
	export default {
		name: 'PathSelect',
		components: {},
		props: {
			alistId: {
				type: Number,
				default: null
			}
		},
		data() {
			return {
				dialogShow: false,
				pathLoading: false,
				submitLoading: false,
				cuPath: null,
				props: {
					label: 'label',
					children: 'child',
					isLeaf: 'leaf'
				}
			};
		},
		created() {},
		beforeDestroy() {},
		methods: {
			show() {
				this.dialogShow = true;
			},
			async getPath(path) {
				this.pathLoading = true;
				try {
					let res = await alistGetPath(this.alistId, path);
					this.pathLoading = false;
					return res; // 返回整个响应对象，包含data和has_more_children
				} catch (err) {
					this.pathLoading = false;
					console.error("Error getting path:", err);
					return { data: { data: [], has_more_children: false }, msg: 'Error' }; // 返回默认结构，模拟后端完整响应
				}
			},
			async loadNode(node, resolve) {
				let currentPath = '/';
				if (node.level !== 0) {
					currentPath = node.data.path;
				}
				
				console.log('DEBUG: loadNode called for path:', currentPath, 'Node:', node); // DEBUG
				let response = await this.getPath(currentPath);
				console.log('DEBUG: getPath raw response:', response); // DEBUG

				let nodes = [];
				// 确保正确访问 response.data.data
				const backendData = response && response.data ? response.data.data : [];
				const hasMoreChildren = response && response.data ? response.data.has_more_children : false;

				console.log('DEBUG: Extracted backendData:', backendData); // DEBUG
				console.log('DEBUG: Extracted hasMoreChildren:', hasMoreChildren); // DEBUG

				if (backendData && backendData.length > 0) {
					nodes = backendData.map(item => {
						let fullPath = currentPath + (currentPath.endsWith('/') ? '' : '/') + item.path;
						return {
							path: fullPath,
							label: item.path,
							isLeaf: false, // 明确设置为非叶子节点，以便el-tree显示展开图标
						};
					});
				}

				// 如果后端指示还有更多子节点，添加一个提示
				if (hasMoreChildren) {
					nodes.push({
						path: currentPath, // 路径保持当前路径，不影响选择
						label: '（该目录下还有更多未显示项目，请手动输入或搜索）',
						isLeaf: true, // 标记为叶子节点，不可展开
						disabled: true // 禁用选择
					});
				}
				console.log('DEBUG: Final nodes to resolve:', nodes); // DEBUG
				resolve(nodes);
			},
			closeShow() {
				this.dialogShow = false;
				this.cuPath = null;
			},
			nodeClick(dt, node, se) {
				// 避免选择提示节点
				if (node.disabled) {
					this.cuPath = null;
					return;
				}
				this.cuPath = node.data.path;
			},
			submit() {
				this.$emit('submit', this.cuPath);
				this.closeShow();
			}
		}
	}
</script>

<style lang="scss" scoped>
	.pathSelect {}
</style>